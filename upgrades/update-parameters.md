# Updating `OPSuccinctL2OutputOracle` Parameters

OP Succinct supports a rolling update process when [program binaries](https://succinctlabs.github.io/op-succinct/advanced/verify-binaries.html) must be reproduced and only the `aggregationVkey`, `rangeVkeyCommitment` or `rollupConfigHash` parameters change. For example, this could happen if

-   The SP1 version changes
-   An optimization to the range program is released
-   Some L2 parameters change

## Rolling update guide

1. Generate new elfs, vkeys, and a rollup config hash by following [this guide](https://succinctlabs.github.io/op-succinct/advanced/verify-binaries.html).
2. Run the procedure to add a new config to the chain

1. Spin up a new proposer with the [`OP_SUCCINCT_CONFIG_NAME`](https://succinctlabs.github.io/op-succinct/proposer.html#optional-environment-variables) environment variable set to the name of the config you added. For this example, you would set `OP_SUCCINCT_CONFIG_NAME="my_upgrade"` in your `.env` file.
2. Spin up a new prover pointing to the new proposer.
3. Pre-configure Aggsender to point to the new prover but don't restart yet.
4. Stop the Aggsender
5. [Update the selected config]() in the contracts
6. Start the Aggsender
3. For security, delete your old `OpSuccinctConfig` by running the procedure in [this runbook]()
