# CDK FEP to PP Upgrade

This document provides a comprehensive guide for upgrading from CDK Erigon FEP (Fork 12 or 13) to PP (Pessimistic Proofs) mode.

- [CDK FEP to PP Upgrade](#cdk-fep-to-pp-upgrade)
  - [Key Changes](#key-changes)
  - [Prerequisites](#prerequisites)
  - [Upgrade procedure](#upgrade-procedure)

## Key Changes
- Proof generation moved from trusted infrastructure to agglayer-prover (run by Polygon).
- Neither pool-manager nor executors required to be run in the trusted infrastructure.
- Changing cdk-node component for aggkit to verify batches and submit certificates to the Agglayer.

## Prerequisites

Deploy Aggkit in sync only mode.

* image: ghcr.io/agglayer/aggkit:0.4.0
* command: aggkit run --cfg=/app/config/config.toml --components=l1infotreesync
* persisted data: /app/data
* No environment variables

Also, create the config.toml (TBD):
```toml
PathRWData = "/app/data"

L1URL = "${L1_URL}"
L2URL = "${L2_URL}"

AggLayerURL = "${AGGLAYER_URL}"

NetworkID = "${L2_CHAINID}"
SequencerPrivateKeyPath = "/app/config/sequencer.keystore"
SequencerPrivateKeyPassword = "${SEQ_KEYSTORE_PASSW}"

polygonBridgeAddr = "${BRIDGE_ADDR}"

rollupCreationBlockNumber = "${R_BLOCKNUMBER}"
rollupManagerCreationBlockNumber = "${RM_BLOCKNUMBER}"
genesisBlockNumber = "${GENESIS_BLOCKNUMBER}"

[Log]
Environment = "production"
Level = "${LOG_LEVEL}"
Outputs = ["stdout"]

[L1Config]
chainId = "${L1_CHAINID}"
polygonZkEVMAddress = "${ROLLUP_ADDR}"
polygonRollupManagerAddress = "${ROLLUP_MANAGER_ADDR}"
polygonZkEVMGlobalExitRootAddress = "${GER_ADDR}"
polTokenAddress = "${POL_ADDR}"

[L2Config]
GlobalExitRootAddr = "${GER_ADDR}"

[L1InfoTreeSync]
SyncBlockChunkSize = 1000
BlockFinality = "FinalizedBlock"
InitialBlock = "${R_BLOCKNUMBER}"

[AggSender]
CertificateSendInterval = "1m"
UseAgglayerTLS = true
MaxCertSize = 0
```

Once started, it will sync from the rollup manager deployment block. It may take a few hours to complete.

> [!NOTE]
> L2_URL requires **debug_** methods enabled.

## Upgrade procedure

This process may take a couple hours to complete, but downtime from the point of view of the users should be a simple restart. Ensure the aggkit is fully synced with the lastes block on L1.

1. **Stop sequencing**: Stop the sequencer-sender component.
2. **Wait for verficitation**: Wait till the aggregator verify all sequenced batches. Wait until the last verification transaction is finalized.
3. **Update components**:
   1. Update erigon version to _hermeznetwork/cdk-erigon:v2.61.23_
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
   1. Send the transaction to perform the upgrade: `cast send --private-key ${ADMIN_PKEY} $ROLLUP_MANAGER "initMigration(uint32,uint32, bytes)" ${ROLLUPID} ${ROLLUPTYPEID} 0x`
   2. Wait until the transaction is finalized.
5. **Start aggsender**:
   1. Update aggkit command: `aggkit run --cfg=/app/config/config.toml --components=aggsender`
   2. Monitor the first certificate is correctly sent to the agglayer.
