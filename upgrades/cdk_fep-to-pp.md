# CDK FEP to PP Upgrade

This document provides a comprehensive guide for upgrading from CDK Erigon FEP (Fork 12 or 13) to PP (Pessimistic Proofs) mode.

- [CDK FEP to PP Upgrade](#cdk-fep-to-pp-upgrade)
  - [Key Changes](#key-changes)
  - [Prerequisites](#prerequisites)
  - [Upgrade procedure](#upgrade-procedure)

## Key Changes
- Proof generation moved from trusted infrastructure to agglayer-prover.
- No pool-manager or executors required from erigon trusted infrastructure.
- Changing cdk-node component for aggkit to verify batches.

## Prerequisites

Deploy Aggkit in sync only mode. 

* image: ghcr.io/agglayer/aggkit:0.4.0
* command: aggkit run --cfg=/app/config/config.toml --components=l1infotreesync
* persistance disk: /app/data
* No environment variables

Also, create the config.toml (TBD):
```toml
AggLayerURL = "${AGGLAYER_URL}"
NetworkID = ${L2_CHAINID}
PathRWData = "/app/data"
L1URL = "${L1_URL}"
L2URL = "${L2_URL}"
ContractVersions = "${CONTRACT_VERSIONS}"
IsValidiumMode = false
L2Coinbase = "${POL_ADDR}"
SequencerPrivateKeyPath = "/app/config/sequencer.keystore"
SequencerPrivateKeyPassword = "${SEQ_KEYSTORE_PASSW}"
AggregatorPrivateKeyPath = ""
AggregatorPrivateKeyPassword  = ""
SenderProofToL1Addr = ""
polygonBridgeAddr = "${BRIDGE_ADDR}"
RPCURL = "${L2_URL}"
WitnessURL = ""
rollupCreationBlockNumber = "${R_BLOCKNUMBER}"
rollupManagerCreationBlockNumber = "${RM_BLOCKNUMBER}"
genesisBlockNumber = "${GENESIS_BLOCKNUMBER}"
AggsenderPrivateKey = {Path = "/app/config/sequencer.keystore", Password = "${SEQ_KEYSTORE_PASSW}"}

[Log]
Environment = "production"
Level = "${LOG_LEVEL}"
Outputs = ["stdout"]

[L1Config]
chainId = "${L1_CHAINID}"
polygonZkEVMGlobalExitRootAddress = "${GER_ADDR}"
polygonRollupManagerAddress = "${ROLLUP_MANAGER_ADDR}"
polTokenAddress = "${POL_ADDR}"
polygonZkEVMAddress = "${ROLLUP_ADDR}"

[L2Config]
GlobalExitRootAddr = "${GER_ADDR}"

[RPC]
Port = 5576

[AggSender]
AggsenderPrivateKey = {Path = "/app/config/sequencer.keystore", Password = "${SEQ_KEYSTORE_PASSW}"}
CertificateSendInterval = "1m"
CheckSettledInterval = "5s"
MaxCertSize = 0
SaveCertificatesToFilesPath = "/app/data"
BlockFinality="FinalizedBlock"
Mode="PessimisticProof"
RequireNoFEPBlockGap = true
UseAgglayerTLS = true

[L1InfoTreeSync]
BlockFinality="FinalizedBlock"
SyncBlockChunkSize=1000
InitialBlock = ${R_BLOCKNUMBER}
```

Once started, it will sync from the rollup manager deployment block. It may take some hours to complete.

> [!NOTE]
> Note that L2_URL requieres **debug_** methods enabled.

## Upgrade procedure

This process may take a couple hours to complete, but downtime from the point of view of the users should be ~30min. Ensure the aggkit is fully synced with the lastes block on L1. I also recommend to increase provers a couple hours before starting the procedure to minimize the trusted_verified batch gap.

1. **Halt Sequencer**: (~15min)
   1. Get the current batch from the sequencer: `cast rpc zkevm_batchNumber`
   2. Edit the sequencer command to add the following parameter -> `--zkevm.sequencer-halt-on-batch-number=$batch+2`
   3. Restart the sequencer to apply this new parameter.
   4. Wait (should take a bit over the batch_time configured on zkevm.sequencer-batch-seal-time) until the seqeuncer show logs about the Halt on the specified batch.
2. **Wait for verficitation**: (~1h)
   1. Sequencer is halted on `$batch+1`. So, we need to wait till the sequence-sender includes this batch on a sequence and the aggregator generates the proof and send everything to L1. We need to wait until the verification transaction is finalized.
3. **Stop all components**: (~15min)
   1. Once the las verification tx is finalized, stop all components.
   2. (optional) Backup erigon datadir.
4. **Send upgrade Tx**:(~15min)
   1. Send the transaction to perform the upgrade: `cast send --private-key ${ADMIN_PKEY} $ROLLUP_MANAGER "initMigration(uint32,uint32, bytes)" ${ROLLUPID} ${ROLLUPTYPEID} 0x`
   2. Wait until the transaction is finalized (you can proceed with next step, but do not start any component yet).
5. **Update configurations**: (~15min)
   1. Update to version X (TBD - for forkid compatibility) 
   2. Remove halt parameter on sequencer command.
   3. Update sequencer config with:
      ```yaml
      # zkevm.executor-urls: "${STATELESS_EXECUTOR}" # Remove executors
      zkevm.executor-strict: false
      zkevm.disable-virtual-counters: true
      zkevm.mock-witness-generation: true
      ```
   4. Update RPC config with:
      ```yaml
      # zkevm.pool-manager-url: "${POOL_MANAGER_URL}" # Remove pool-manager-url
      zkevm.mock-witness-generation: true
      zkevm.disable-virtual-counters: true
      ```
   5. Ensure the aggkit has the agglayer_url for GRPC, not the HTTPS.
   6. Update aggkit command: aggkit run --cfg=/app/config/config.toml --components=aggsender
   7. Disable/Remove the following components.
      1. prover
      2. executor
      3. pool-manager
      4. cdk-node
      5. dac
6. **Start infrastructure**: (~15min)
   1. Ensure the upgrade transaction is finalized
   2. Start the sequencer and ensure that new blocks are being generated. Ensure the new blocks still are on your previous fork id (TBD).
   3. Start RPC and other erigon instances.
   4. Start aggkit
   5. Start bridge, explorer and other components.
