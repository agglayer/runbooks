# FEP Rollup upgrades

> [!WARNING]
> Only AMD64 (linux/amd64) Docker images have been tested and are guaranteed to be supported. Other architectures are not officially supported.

OP Succinct supports a rolling update process when [program binaries](https://succinctlabs.github.io/op-succinct/advanced/verify-binaries.html) must be reproduced and only the `aggregationVkey`, `rangeVkeyCommitment`, or `rollupConfigHash` parameters change. For example, this could happen if:
* The SP1 version changes
* An optimization to the range program is released
* Some L2 parameters change

> [!NOTE]
> In case you're migrating from [v2 to v3](v2-to-v3.md) check the guide for configuration changes

> [!NOTE]
> In case you're migrating from [v3.1 to v3.4](v3_1_to_v3_4.md) check the guide for configuration changes

---

## Update guide

> [!IMPORTANT]
> The **Aggchain manager** address is required. This will typically be a multisig or timelock.

> [!IMPORTANT]
> The `_configName` in the contract is a `bytes32` value, derived from hashing the human-readable config name string with `keccak256`.

* Example:

  ```
  keccak256("opsuccinct_genesis")
  ```

  represents the config name `"opsuccinct_genesis"`.

---
### Setup

Export env vars
```shell
export rollup_address=0x
export l1_rpc_url=https://
export l2_node_url=https://
export l2_rpc_url=https://
export op_succinct_version=[replace with the target version]
```

Confirm if you control the aggchainManager address
```shell
cast call $rollup_address "aggchainManager()(address)" --rpc-url $l1_rpc_url
```

Grab the current configName for future reference
```shell
cast call $rollup_address "selectedOpSuccinctConfigName()(bytes32)" --rpc-url $l1_rpc_url
```

---

### Add a new config on-chain

Collect the necessary info

```shell
mkdir upgrade
```

```shell
cd upgrade
```

```shell
cat << EOF > .env
L1_RPC="${l1_rpc_url}"
L1_BEACON_RPC="${l1_rpc_url}"
L2_NODE_RPC="${l2_node_url}"
L2_RPC="${l2_rpc_url}"
EOF
```

Run the appropriate command based on your version:

**For versions <= v3.1.0-agglayer:**
```shell
docker run --rm -it \
  --platform linux/amd64 \
  -v "$(pwd)":/tmp/env \
  ghcr.io/agglayer/op-succinct/op-succinct:${op_succinct_version} \
  fetch-l2oo-config \
  --env-file /tmp/env/.env \
  --output-dir /tmp/env
```

**For versions >= v3.3.0-agglayer:**
```shell
docker run --rm -it \
  --env OP_SUCCINCT_L2_OUTPUT_ORACLE_CONFIG_PATH=/tmp/env/opsuccinctl2ooconfig.json \
  --platform linux/amd64 \
  -v "$(pwd)":/tmp/env \
  ghcr.io/agglayer/op-succinct/op-succinct-agglayer:${op_succinct_version} \
  /bin/bash -c "fetch-l2oo-config --env-file /tmp/env/.env"
```

The command will output a `opsuccinctl2ooconfig.json` file that looks something like this:
```json
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
```shell
cast keccak "${op_succinct_version}"
```

Using the **Aggchain manager** address, call:

```solidity
function addOpSuccinctConfig(
    bytes32 _configName,
    bytes32 _rollupConfigHash,
    bytes32 _aggregationVkey,
    bytes32 _rangeVkeyCommitment
)
```

Cast command:
```shell
cast send $rollup_address \
  "addOpSuccinctConfig(bytes32,bytes32,bytes32,bytes32)" \
  <_configName> \
  <_rollupConfigHash> \
  <_aggregationVkey> \
  <_rangeVkeyCommitment> \
  --rpc-url $l1_rpc_url \
  --private-key $PRIVATE_KEY
```

This method can be found in the rollup contract.

---

### Direct Update
1. Stop the **aggsender**, **aggkit-prover** and **op-succinct-proposer**
2. Adjust the [**op-succinct-proposer**](#op-succinct-proposer) with the new `OP_SUCCINCT_CONFIG_NAME` environment variable and version
3. Adjust the [**aggkit-prover**](#aggkit-prover) and [**aggsender**](#aggsender) config files and versions
4. Update the selected config on-chain using the **Aggchain manager**:

```solidity
function selectOpSuccinctConfig(
    bytes32 _configName
)
```

Cast command:
```shell
cast send $rollup_address \
  "selectOpSuccinctConfig(bytes32)" \
  <_configName> \
  --rpc-url $l1_rpc_url \
  --private-key $PRIVATE_KEY
```

5. Start the **aggkit-prover**, **op-succinct-proposer**. Wait until the services look stable
6. Start the **aggsender**

---

### Cleanup (security step)

For security, delete the old config by calling the **Aggchain manager**:

```solidity 
function deleteOpSuccinctConfig(
    bytes32 _configName
) external onlyAggchainManager
```

Cast command:
```shell
cast send $rollup_address \
  "deleteOpSuccinctConfig(bytes32)" \
  <_configName> \
  --rpc-url $l1_rpc_url \
  --private-key $PRIVATE_KEY
```
