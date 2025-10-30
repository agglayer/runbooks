# CDK Erigon FEP to PP Upgrade

This document provides a comprehensive guide for upgrading from CDK Erigon FEP (Fork 12 or 13) to PP (Pessimistic Proofs) mode.

## Key Changes
- Proof generation moved from trusted infrastructure to agglayer-prover (run by Polygon).
- Neither pool-manager nor executors required to be run in the trusted infrastructure.
- Changing cdk-node component for aggkit to verify batches and submit certificates to the Agglayer.

## Prerequisites

Deploy Aggkit in sync only mode.

* image: ghcr.io/agglayer/aggkit:0.7.0
* command: aggkit run --cfg=/etc/aggkit/config.toml --components=bridge
* Persisted data directory. Ex.: /data
* Configuration file. Ex: /etc/aggkit/config.toml
* No environment variables

Also, create the config.toml file with the following template:
<details>
<summary>config.toml template</summary>
  
```toml
NetworkID = 60                       # Network id
PathRWData = "/data"                 # Persistent directory
L1URL = "https://..."                # L1_URL
L2URL = "https://..."                # L2_URL
polygonBridgeAddr = "0x..."          # Bridge SC address
rollupCreationBlockNumber = 0        # Rollup SC deployment block
rollupManagerCreationBlockNumber = 0 # Rollup Mananger SC deployment block
genesisBlockNumber = 0               # Rollup Mananger SC deployment block

[L1Config]
chainId = 11155111                          # L1 chain id
polygonZkEVMGlobalExitRootAddress = "0x..." # L1 GER SC address
polygonRollupManagerAddress = "0x..."       # L1 Rollup Manager SC address
polTokenAddress = "0x..."                   # L1 POL token SC address
polygonZkEVMAddress = "0x..."               # L1 Rollup SC address

[L2Config]
GlobalExitRootAddr = "0x..." # L2 GER SC address

[AggSender]
AggsenderPrivateKey = {Path = "/etc/aggkit/sequencer.keystore", Password = "XXXX"}
CertificateSendInterval =        #
CheckSettledInterval =           #
MaxCertSize =                    #
SaveCertificatesToFilesPath = "" # 
Mode=""                          #
RequireNoFEPBlockGap = true      #
DryRun = false                   #
MaxL2BlockNumber = 0             #

```
</details>

Once started, it will sync from the rollup manager deployment block. It may take a few hours to complete.

> [!NOTE]
> L2_URL requires **debug_** methods enabled.

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
   1. Request Polygon (as the RollupManager Admin) to send the transaction to perform the migration: `cast send --private-key ${ADMIN_PKEY} $ROLLUP_MANAGER "initMigration(uint32,uint32, bytes)" ${ROLLUPID} ${ROLLUPTYPEID} 0x`
   2. Wait until the transaction is finalized.
5. **Start aggsender**:
   1. Get last l2 block verified:
      1. Set the correct ETH_RPC_URL for your network: `export ETH_RPC_URL="https://zkevm-rpc.com"`
      2. Get the last verified batch number: `cast rpc zkevm_verifiedBatchNumber`
      3. Get the last block hash from previous batch: `cast rpc zkevm_getBatchByNumber $(cast rpc zkevm_verifiedBatchNumber) --json | jq -r .blocks[-1]`
      4. Get the block number from previous block hash: `cast rpc eth_getBlockByHash $(cast rpc zkevm_getBatchByNumber $(cast rpc zkevm_verifiedBatchNumber) --json | jq -r .blocks[-1]) | jq -r .number`
      5. Convert the block number from HEX to DEC: `printf "%d\n" $(cast rpc eth_getBlockByHash $(cast rpc zkevm_getBatchByNumber $(cast rpc zkevm_verifiedBatchNumber) --json | jq -r .blocks[-1]) | jq -r .number)`
   3. Update aggkit config:
      ```toml
      [AggSender]
      MaxL2BlockNumber = 0 # <- Set the last verified L2 block number
      ```
   4. Update aggkit command: `aggkit run --cfg=/app/config/config.toml --components=aggsender,bridge`
   5. Start the aggkit instance with the new config and command changes.
   6. Monitor the first certificate is correctly sent to the agglayer.
   7. Once the first certificate is settled, you can rollback configuration change.
      ```toml
      [AggSender]
      MaxL2BlockNumber = 0
      ```
