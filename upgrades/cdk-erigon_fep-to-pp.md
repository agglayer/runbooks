# CDK Erigon FEP to PP Upgrade

This document provides a comprehensive guide for upgrading from CDK Erigon FEP (Fork 12 or 13) to PP (Pessimistic Proofs) mode.

## Key Changes
- Proof generation moved from trusted infrastructure to agglayer-prover (run by Polygon).
- Neither pool-manager nor executors required to be run in the trusted infrastructure.
- Changing cdk-node component for aggkit to verify batches and submit certificates to the Agglayer.

> [!WARNING]
> **Following this runbook drops the network's data-availability (DA) capabilities.** A PP
> network no longer posts batch data to L1 (calldata/blobs) nor to a DAC — those are rollup
> features that are lost when switching to PP. As a consequence the **sequencer becomes a single
> point of failure**: if the operator does not keep proper backups, replica nodes, etc., the L2
> data cannot be reconstructed.

## Prerequisites

### Deploy Aggkit in sync only mode

Deploy Aggkit in sync only mode.

* image: ghcr.io/agglayer/aggkit:0.8.1
* command: aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender
* Persisted data directory. Ex.: /data
* Configuration file. Ex: /etc/aggkit/config.toml
* No environment variables

Also, create the config.toml file with the following template:
<details>
<summary>config.toml template</summary>
  
```toml
PathRWData = "/data"                 # Persistent directory
L1URL = "https://..."                # L1_URL
L2URL = "https://..."                # L2_URL
rollupCreationBlockNumber = 0        # Rollup SC deployment block
rollupManagerCreationBlockNumber = 0 # Rollup Mananger SC deployment block
genesisBlockNumber = 0               # Rollup Mananger SC deployment block

[L1Config]
BridgeAddr = "0x..."                        # L1 bridge contract address
chainId = 11155111                          # L1 chain id
polygonZkEVMGlobalExitRootAddress = "0x..." # L1 GER SC address
polygonRollupManagerAddress = "0x..."       # L1 Rollup Manager SC address
polTokenAddress = "0x..."                   # L1 POL token SC address
polygonZkEVMAddress = "0x..."               # L1 Rollup SC address

[L2Config]
BridgeAddr = "0x..."         # L2 bridge contract address
GlobalExitRootAddr = "0x..." # L2 GER SC address

[AggSender]
AggsenderPrivateKey = {Path = "/etc/aggkit/sequencer.keystore", Password = "XXXX"}
CertificateSendInterval = "1m"
CheckStatusCertificateInterval = "1m"
CheckSettledInterval = "5s"
RetryCertAfterInError = true
SaveCertificatesToFilesPath = "/tmp"
RequireNoFEPBlockGap = true
MaxL2BlockNumber = 0
MaxCertSize = 0
DryRun = true
  [AggSender.AgglayerClient.GRPC]
  URL = "grpc-agglayer.polygon.technology:443" # It depends on the environment.
  UseTLS = "true"
```
</details>

Once started, it will sync from the rollup manager deployment block. It may take a few hours to complete.

> [!NOTE]
> L2_URL requires **debug_** methods enabled.

> [!NOTE]
> AGGLAYER GRPC URL depends on the environmnet
> * mainnet: "grpc-agglayer.polygon.technology:443"
> * cardona: "grpc-agglayer-test.polygon.technology:443"
> * bali: "grpc-agglayer-dev.polygon.technology:443"

### Reconcile pending batches (`lastBatchSequenced` vs `lastVerifiedBatch`)

> [!IMPORTANT]
> The migration transaction `initMigration` (see step 4 of the Upgrade procedure) **only succeeds
> when `lastBatchSequenced == lastVerifiedBatch`**. Run this pre-flight check first — if there are
> sequenced-but-unverified batches, the migration will revert until the gap is closed.

Read both counters from the **Rollup Manager** on L1:

```bash
export ETH_RPC_URL="https://..."      # L1 RPC
export ROLLUP_MANAGER="0x..."          # L1 Rollup Manager SC
export ROLLUP="0x..."                  # L1 Rollup SC
export ROLLUP_ID=<n>                    # NetworkID / rollupID

# lastVerifiedBatch — dedicated getter on the Rollup Manager:
cast call $ROLLUP_MANAGER "getLastVerifiedBatch(uint32)(uint64)" $ROLLUP_ID

# lastBatchSequenced — field 6 of the rollup-data tuple on the Rollup Manager:
cast call $ROLLUP_MANAGER \
  "rollupIDToRollupData(uint32)(address,uint64,address,uint64,bytes32,uint64,uint64,uint64,uint64,uint64,uint64,uint8)" \
  $ROLLUP_ID | sed -n '6p'
```

If `lastBatchSequenced == lastVerifiedBatch`, there is nothing to reconcile — continue with the
Upgrade procedure.

If `lastBatchSequenced > lastVerifiedBatch`, there are sequenced batches that were never verified.
You must close the gap **before** migrating by rolling the pending batches back.

#### Rollback the pending batches (`rollbackBatches`)

Discard the sequenced-but-unverified batches so both counters line up at
`lastVerifiedBatch`. `rollbackBatches(IPolygonRollupBase rollupContract, uint64 targetBatch)` on the
Rollup Manager is a single L1 transaction (callable by `_UPDATE_ROLLUP_ROLE` or the rollup admin)
that rewinds `lastBatchSequenced`, `totalSequencedBatches`, the `sequencedBatches` entries and
`lastAccInputHash`; it leaves `lastLocalExitRoot` and `lastVerifiedBatch` untouched, so the bootstrap
certificate still targets the same LER.

1. **Stop the sequencer** so no new batches are sequenced during the rollback and after the rollback.
2. **Trigger the rollback** on L1 (`targetBatch = lastVerifiedBatch`):
   ```bash
   cast send --private-key $ADMIN_PKEY $ROLLUP_MANAGER \
     "rollbackBatches(address,uint64)" $ROLLUP $LAST_VERIFIED_BATCH
   ```
   Wait until the transaction is finalized, then re-check that `lastBatchSequenced == lastVerifiedBatch`.

Then continue with the Upgrade procedure below.

> [!NOTE]
> cdk-erigon does **not** drop L2 blocks when it observes the `RollbackBatches` event on L1, so the
> L2 chain is not reorged by the rollback and blocks after `lastVerifiedBatch` remain; their bridge
> exits settle in the PP certificates after the migration. Validate this on a shadow fork before
> running it against a live network.

## Upgrade procedure

This process may take a couple hours to complete, but downtime from the point of view of the users should be equivalent to a simple node restart. Ensure the aggkit is fully synced with the latest block on L1.

1. **Stop sequencing**: Stop the sequencer-sender component.
2. **Wait for verification**: Wait until the aggregator verifies all sequenced batches. Wait until the last verification transaction is finalized.
3. **Update components**:
   1. Update erigon version to _hermeznetwork/cdk-erigon:v2.61.24_ or newer.
   2. Update sequencer config with:
      ```yaml
      # zkevm.executor-urls: "${STATELESS_EXECUTOR}" # Remove executors
      zkevm.executor-strict: false
      zkevm.disable-virtual-counters: true
      zkevm.mock-witness-generation: true
      ```
   3. Update RPC config with:
      ```yaml
      # zkevm.pool-manager-url: "${POOL_MANAGER_URL}" # Remove pool-manager-url
      zkevm.mock-witness-generation: true
      zkevm.disable-virtual-counters: true
      ```
   4. Stop the following components:
      1. dac
      2. sequence-sender
      3. aggregator
      4. executors
      5. provers
      6. pool-manager
4. **Migrate to PP**:
   1. Request Polygon (as the AgglayerManager Admin) to send the transaction to perform the migration: `cast send --private-key ${ADMIN_PKEY} $AGGLAYER_MANAGER "initMigration(uint32,uint32,bytes)" ${ROLLUPID} ${ROLLUPTYPEID} 0x06e76665`
      1. Rollup ID of the network.
      2. Rolluptype should be latest AggchainECDSAMultisig.
      3. Data should be initialized with `cast calldata "migrateFromLegacyConsensus()"`, `0x06e76665`. This is the function used to migrate the Aggchain from PolygonPessimisticConsensus or PolygonRollupBaseEtrog to AggchainECDSAMultisig. Specifically, this function will update the Aggchain state with the following changes:
         1. Preserve existing admin.
         2. Set `_initializerVersion = 1`. (aggchainECDSAMultisig)
         3. Set `threshold = 1` and add `trustedSequencer` as the sole initial signer. Admin can later update signers and threshold via `updateSignersAndThreshold`.
         4. Handles empty `trustedSequencerURL` by using "NO_URL" placeholder.
   2. Wait until the transaction is finalized.
7. **Start aggsender**:
   1. Get last l2 block verified:
      1. Set the correct ETH_RPC_URL for your network: `export ETH_RPC_URL="https://zkevm-rpc.com"`
      2. Get the last verified batch number: `cast rpc zkevm_verifiedBatchNumber`
      3. Get the last block hash from previous batch: `cast rpc zkevm_getBatchByNumber $(cast rpc zkevm_verifiedBatchNumber) --json | jq -r .blocks[-1]`
      4. Get the block number from previous block hash: `cast rpc eth_getBlockByHash $(cast rpc zkevm_getBatchByNumber $(cast rpc zkevm_verifiedBatchNumber) --json | jq -r .blocks[-1]) | jq -r .number`
      5. Convert the block number from HEX to DEC: `printf "%d\n" $(cast rpc eth_getBlockByHash $(cast rpc zkevm_getBatchByNumber $(cast rpc zkevm_verifiedBatchNumber) --json | jq -r .blocks[-1]) | jq -r .number)`
   3. Update aggkit config:
      ```toml
      [AggSender]
      MaxL2BlockNumber = 0 # Set the obtained last verified L2 block number
      DryRun = false       # Send certificate to the agglayer
      MaxCertSize = 0      # Do not cap the certificate size (default is 8MB)
      MaxL2BlockRange = 0  # Do not cap the block range (already the default value)
      ```
      > [!IMPORTANT]
      > To guarantee the **bootstrap certificate covers the full `[1, N]` range** (where `N` is the
      > last verified L2 block) **in a single, non-split certificate**, both of these limits must be
      > disabled:
      > * `MaxCertSize = 0` — otherwise the default 8MB cap can split the bootstrap cert.
      > * `MaxL2BlockRange = 0` — this is already the default, but set it explicitly to be safe.
   4. Restart the aggkit instance with the new config.
   5. Monitor the first certificate is correctly sent to the agglayer.
   6. Once the first certificate is settled, update the configuration to allow new certificates.
      ```toml
      [AggSender]
      MaxL2BlockNumber = 0
      ```
