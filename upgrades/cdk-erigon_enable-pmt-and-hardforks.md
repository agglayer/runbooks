# CDK Erigon enable PMT and ETH hardforks

This document provides a comprehensive guide for upgrading CDK Erigon in PP mode to disable GER injection, enable PMT and Ethereum hardforks.

## Key Changes
- Erigon will be compatible with EVM and will enable hardforks.
- Erigon will not perform the GER injection.
- Aggkit aggoracle will be enabled to perform fast GER injection.

## Prerequisites

All erigon instances should be on the version +v2.64.0. In case your networks has an older one, you need to follow this procedure:

1. **Sync new instance**: Create a permissionless instance with new version and let it synchronize.
2. **Halt sequencer**: 
   1. Get the last batch from the sequencer: `cast rpc zkevm_batchNumber`
   2. Add in the sequencer config: `zkevm.sequencer-halt-on-batch-number: $BATCH+2`
   3. Restart sequencer and wait ~10min until you see logs like `Halting sequencer on batch $BATCH+2...`
3. **Stop all instances**: Stop all instances, including datastream, RPCs, sequencer, etc.
4. **Replace datadir**: Copy the datadir from the new synced instance and replace it everywhere.
5. **Start sequencer**: Remove the sequencer's halt config and start it with the new datadir.
6. **Start RPCs**: Once the sequencer is generating new blocks, start the rest of the instances.
7. **Update permissionless**: (Optionally) Ask your partners and anyone running a permissionless instance to update to the same version as the trusted infrastructure.

> [!NOTE]
> This procedure produces full downtime between steps 3 and 6, which using k8s snapshots in the zkEVM takes about 30 minutes. But it's recommended to test in your own network before proceeding to have a clear estimation.

Also, The new component, the aggoracle, needs a wallet with funds on L2. You can use a new or existing wallet, you could to bridge some funds or directly transfer them from another wallet. Once you have the wallet ready, you need to update the SC acordingly. To do that, (TBD - Carlos).

## Upgrade procedure

This process may take under 30 minutes to be completed, but downtime from the point of view of the users should be equivalent to a simple node restart. Ensure the aggkit is fully synced with the latest block on L1.

1. **Sync new instance**: Create a permissionless instance adding the configuration `zkevm.simultaneous-pmt-and-smt: true` and let it synchronize.
2. **Halt sequencer**: 
   1. Get the last batch from the sequencer: `cast rpc zkevm_batchNumber`
   2. Add in the sequencer config: `zkevm.sequencer-halt-on-batch-number: $BATCH+2`
   3. Restart sequencer and wait ~10min until you see logs like `Halting sequencer on batch $BATCH+2...`
3. **Stop all instances**: Stop all instances, including datastream, RPCs, sequencer, etc.
4. **Replace datadir**: Copy the datadir from the new synced instance and replace it everywhere.
5. **Update configuration**: Update the following configuration on all instances.
   1. **chainspec.json**:
      ```json
      {
         "pmtEnabledBlock": "$L2BLOCK+1",
         "normalcyBlock": "$L2BLOCK+1",
         "sovereignModeBlock": "$L2BLOCK+1",
         "berlinBlock": "$L2BLOCK+1",
         "londonBlock": "$L2BLOCK+1",
         "shanghaiTime": "$L2BLOCK_TIMESTAMP+1",
         "cancunTime": "$L2BLOCK_TIMESTAMP+1",
         "pragueTime": "$L2BLOCK_TIMESTAMP+1",
      }
      ```
   2. **erigon.yaml**:
      ```yaml
      override.pmtEnabledBlock: $L2BLOCK+1
      override.normalcyBlock: $L2BLOCK+1
      override.sovereignModeBlock: $L2BLOCK+1
      override.prague: $L2BLOCK+1
      override.cancun: $L2BLOCK+1
      override.normalcyblock: $L2BLOCK+1
      override.shanghaitime: $L2BLOCK_TIMESTAMP+1
      override.londonblock: $L2BLOCK_TIMESTAMP+1
      override.pmtenabledblock: $L2BLOCK_TIMESTAMP+1
      zkevm.witness-full: false
      zkevm.executor-strict: false
      zkevm.disable-virtual-counters: true
      zkevm.mock-witness-generation: true
      # zkevm.executor-urls <- remove this
      # zkevm.sequencer-halt-on-batch-number <- remove this
      ```
6. **Start sequencer**: Start sequencer with the new configuration and datadir.
7. **Start RPCs**: Once the sequencer is generating new blocks, start the rest of the instances.
8. **Update permissionless**: Request to partners running a permissionless instance to update to the same version and the same configuration as the trusted infrastructure.
9. **Start aggoracle**:
   1. Add the following configuration in the aggkit:
      ```toml
      [AggOracle]
        [AggOracle.EVMSender]
          GlobalExitRootL2 = "{{L2Config.GlobalExitRootAddr}}"
            [AggOracle.EVMSender.EthTxManager]
              PrivateKeys = [{Method =  "local", Path = "/app/keystore/aggoracle.keystore", Password = "XXXX"}]
      [L1InfoTreeSync]
        BlockFinality = "LatestBlock/-6"
      ```
   2. Add in the command the `aggoracle` component.
10. **Test the network**:
    1.  Ensure new blocks are created
    2.  Test that transactions are being processed. (tranfers, SC calls)
    3.  Ensure aggoracle wallet is doing transactions on the L2 GER SC
    4.  Do some bridges to force a new certificate is sent to the agglayer.

> [!NOTE]
> This procedure produces full downtime between steps 3 and 7, which using k8s snapshots in the zkEVM takes about 30 minutes. Apart from that, all permissionless needs to resync from scratch using new version and configuration. So, you may consider the aditional downtime on the permisionless instances.

> [!NOTE]
> Once you finish this procedure, erigon will be able to unwind from $L2BLOCK+1 on, but not before that block. 

> [!NOTE]
> zkEVM has the config files included in the image. So, they only need to update to a valid version.
