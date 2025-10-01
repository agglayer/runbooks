# Updating `OPSuccinctL2OutputOracle` Parameters

OP Succinct supports a rolling update process when [program binaries](https://succinctlabs.github.io/op-succinct/advanced/verify-binaries.html) must be reproduced and only the `aggregationVkey`, `rangeVkeyCommitment`, or `rollupConfigHash` parameters change. For example, this could happen if:
* The SP1 version changes
* An optimization to the range program is released
* Some L2 parameters change

---

## Rolling update guide

⚠️ **Note:** The **Aggchain manager** address is required. This will typically be a multisig or timelock.

⚠️ **Note:** The `_configName` in the contract is a `bytes32` value, derived from hashing the human-readable config name string with `keccak256`.

* Example:

  ```
  keccak256("opsuccinct_genesis")
  ```

  represents the config name `"opsuccinct_genesis"`.

---
### Setup

1. Export env vars
```
export rollup_address=0x
export l1_rpc_url=https://
export l2_node_url=https://
export l2_rpc_url=https://
```

2. Confirm if you control the aggchainManager address
```
cast call $rollup_address "aggchainManager()" --rpc-url $l1_rpc_url
```

3. Grab the current configName for future reference
```
cast call $rollup_address "selectedOpSuccinctConfigName()" --rpc-url $l1_rpc_url
```

---

### Generate new artifacts

1. Generate new elfs, vkeys, and a rollup config hash by following [this guide](https://succinctlabs.github.io/op-succinct/advanced/verify-binaries.html).

---

### Add a new config on-chain

1. Collect the necessary info

```
mkdir upgrade
```

```
cd upgrade
```

```
cat << EOF > .env
L1_RPC="${l1_rpc_url}"
L1_BEACON_RPC="${l1_rpc_url}"
L2_NODE_RPC="${l2_node_url}"
L2_RPC="${l2_rpc_url}"
EOF
```

Change the following command as needed
```
docker run --rm -it \
  --platform linux/amd64 \
  -v "$(pwd)":/tmp/env \
  ghcr.io/agglayer/op-succinct/op-succinct:v3.1.0-agglayer \
  fetch-l2oo-config \
  --env-file /tmp/env/.env \
  --output-dir /tmp/env
```

The command above will output a `opsuccinctl2ooconfig.json` file that looks something like this:
```
{
  "aggregationVkey": "0x00afb45d8064ae10aa6a1793b8f39a24c27268efae2917b5c02950b2377fbf00",
  "challenger": "0x0000000000000000000000000000000000000000",
  "fallbackTimeoutSecs": 1209600,
  "finalizationPeriod": 3600,
  "l2BlockTime": 1,
  "opSuccinctL2OutputOracleImpl": "0x0000000000000000000000000000000000000000",
  "owner": "0x0000000000000000000000000000000000000000",
  "proposer": "0x0000000000000000000000000000000000000000",
  "proxyAdmin": "0x0000000000000000000000000000000000000000",
  "rangeVkeyCommitment": "0x416d710344b6b6fa2a0b1a1445f3d6ba4fdd5ab43f0e863b1c522db20f28ad9b",
  "rollupConfigHash": "0x2bc101c60102df4dc812a68001d2645cd30e8c61b3e1e8517bdef29e3b21f59c",
  "startingBlockNumber": 604920,
  "startingOutputRoot": "0x01b1a2d21f8296e3d850c035e20aa9377bdd9564ee90e045be39fd4034c31187",
  "startingTimestamp": 1759336632,
  "submissionInterval": 10,
  "verifier": "0x397A5f7f3dBd538f23DE225B51f532c34448dA9B"
}
```

Create a `_configName` by hashing an arbitrary string. The convention is to use the `ghcr.io/agglayer/op-succinct/op-succinct` tag. In this case
```
cast keccak "v3.1.0-agglayer"
0x622142ba8035695383551428b698950d3d4a6a53629c90a86d7192cfb221ae4e
```

2. Using the **Aggchain manager** address, call:

```solidity
function addOpSuccinctConfig(
    bytes32 _configName,
    bytes32 _rollupConfigHash,
    bytes32 _aggregationVkey,
    bytes32 _rangeVkeyCommitment
)
```

This method can be found in the rollup contract.

---

### Spin up updated infra

List of components
- [ghcr.io/agglayer/op-succinct/op-succinct:v3.1.0-agglayer](https://github.com/agglayer/op-succinct/pkgs/container/op-succinct%2Fop-succinct/515633556?tag=v3.1.0-agglayer)
- [ghcr.io/agglayer/aggkit-prover:1.4.2](https://github.com/agglayer/provers/pkgs/container/aggkit-prover/530717765?tag=1.4.2)
- [ghcr.io/agglayer/aggkit:0.7.0-beta8](https://github.com/agglayer/aggkit/pkgs/container/aggkit/530710287?tag=0.7.0-beta8)

1. Spin up a **new proposer** with the [`OP_SUCCINCT_CONFIG_NAME`](https://succinctlabs.github.io/op-succinct/proposer.html#optional-environment-variables) environment variable set to the name of the config you added. For this example, you would set `OP_SUCCINCT_CONFIG_NAME="v3.1.0-agglayer"`

2. Start a new **aggkit-prover** pointing to the new proposer. You may need to adjust the config file
```diff
     [aggchain-proof-service.aggchain-proof-builder]
     network-id = 0
+    proving-timeout = "1h"

     [aggchain-proof-service.aggchain-proof-builder.primary-prover.network-prover]
     proving-timeout = "1h"

-    [aggchain-proof-service.aggchain-proof-builder.proving-timeout]
-    secs = 3600
-    nanos = 0
```

3. Pre-configure **Aggsender** to point to the new prover (do not restart yet). For the new version of aggkit, you may need to adjust the config file
```diff
     [AggSender.AggkitProverClient]
-    URL = https://old-prover-url:port
+    URL = https://new-prover-url:port
     UseTLS = true
```

```diff
     [AggSender]
-    BlockFinality="FinalizedBlock"
-    [AggSender.AgglayerClient]
+    [AggSender.AgglayerClient.GRPC]
     URL = https://
     UseTLS = true
```

> [!NOTE]
> The new aggkit-prover instance will fail with `Error: Unable to setup aggchain proof builder`. This is expected and will be resolved as soon as we finish the upgrade

---

### Switch over

6. Stop the **Aggsender**.

7. Update the selected config on-chain using the **Aggchain manager**:

```solidity
function selectOpSuccinctConfig(
    bytes32 _configName
)
```

8. Start the **Aggsender**.

---

### Cleanup (security step)

9. For security, delete the old config calling using the **Aggchain manager**:

```solidity
function deleteOpSuccinctConfig(
    bytes32 _configName
) external onlyAggchainManager
```
