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

### Generate new artifacts

1. Generate new elfs, vkeys, and a rollup config hash by following [this guide](https://succinctlabs.github.io/op-succinct/advanced/verify-binaries.html).

---

### Add a new config on-chain

2. From the **Aggchain manager**, call:

```solidity
function addOpSuccinctConfig(
    bytes32 _configName,
    bytes32 _rollupConfigHash,
    bytes32 _aggregationVkey,
    bytes32 _rangeVkeyCommitment
)
```

---

### Spin up updated infra

3. Spin up a **new proposer** with the [`OP_SUCCINCT_CONFIG_NAME`](https://succinctlabs.github.io/op-succinct/proposer.html#optional-environment-variables) environment variable set to the name of the config you added. For this example, you would set to the name of the config you added. For this example, you would set `OP_SUCCINCT_CONFIG_NAME="my_upgrade"` in your `.env` file.

4. Start a **new prover** pointing to the new proposer.

5. Pre-configure **Aggsender** to point to the new prover (do not restart yet).

---

### Switch over

6. Stop the **Aggsender**.

7. Update the selected config on-chain using the **Aggchain manager**:

```solidity
function selectOpSuccinctConfig(
    bytes32 _configName
)
```

8. Restart the **Aggsender**.

---

### Cleanup (security step)

9. For security, delete the old config calling using the **Aggchain manager**:

```solidity
function deleteOpSuccinctConfig(
    bytes32 _configName
) external onlyAggchainManager
```
