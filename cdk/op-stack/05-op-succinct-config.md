# OP Succinct Configuration

This document explains how to add and select an op-succinct (L2 output) configuration on-chain using the Aggchain manager.

> **Note:** You need to perform this process for every new version of the op-succinct-proposer component you intend to deploy on your network.

## TL;DR

1. Export RPC endpoints, rollup address, and manager private key
2. Use the op-succinct container to produce `opsuccinctl2ooconfig.json`
3. Call `addOpSuccinctConfig` with the config values
4. Call `selectOpSuccinctConfig` to activate the configuration

## Step 1: Setup Environment Variables

Export the following variables (replace placeholders with your values):

```shell
export rollup_address=0x... # Rollup contract address
export l1_rpc_url="https://<your_l1_rpc>" # L1 RPC endpoint
export l1_beacon_rpc_url="https://<your_l1_beacon_rpc>" # L1 Beacon RPC endpoint
export op_node_url="http://<your_op_node>" # op-node RPC endpoint
export op_reth_url="http://<your_op_reth>" # op-reth RPC endpoint
export aggchain_manager_private_key=0x... # Private key of Aggchain manager
# See Component Versions table in README.md for current values
export op_succinct_version="<op_succinct_version>"
export config_name=$(cast keccak "${op_succinct_version}")
```

## Step 2: Verify Aggchain Manager

Confirm you control the Aggchain manager address stored in the rollup contract:

```shell
cast call $rollup_address "aggchainManager()(address)" --rpc-url $l1_rpc_url
```

Retrieve the currently-selected config name for future reference:

```shell
cast call $rollup_address "selectedOpSuccinctConfigName()(bytes32)" --rpc-url $l1_rpc_url
```

## Step 3: Add New Configuration On-Chain

### 3.1: Create Working Directory

Create a working directory and write a minimal `.env` file used by the op-succinct helper:

```shell
mkdir -p config_${op_succinct_version}
cd config_${op_succinct_version}

cat > .env <<EOF
L1_RPC="${l1_rpc_url}"
L1_BEACON_RPC="${l1_beacon_rpc_url}"
L2_NODE_RPC="${op_node_url}"
L2_RPC="${op_reth_url}"
EOF
```

### 3.2: Generate Configuration File

Run the op-succinct container to fetch/generate the L2 output-oracle configuration:

```shell
docker run --rm -it \
  --env OP_SUCCINCT_L2_OUTPUT_ORACLE_CONFIG_PATH=/tmp/env/opsuccinctl2ooconfig.json \
  --platform linux/amd64 \
  -v "$(pwd)":/tmp/env \
  ghcr.io/agglayer/op-succinct/op-succinct-agglayer:${op_succinct_version} \
  /bin/bash -c "fetch-l2oo-config --env-file /tmp/env/.env"
```

After completion, you should have `opsuccinctl2ooconfig.json` in the working directory containing the required fields.

### 3.3: Add Configuration On-Chain

Call `addOpSuccinctConfig` from the Aggchain manager to publish the new configuration.

**Function signature:**

```solidity
function addOpSuccinctConfig(
    bytes32 _configName,
    bytes32 _rollupConfigHash,
    bytes32 _aggregationVkey,
    bytes32 _rangeVkeyCommitment
)
```

**Example using `cast`** (reads values from `opsuccinctl2ooconfig.json`):

```shell
cast send $rollup_address \
  "addOpSuccinctConfig(bytes32,bytes32,bytes32,bytes32)" \
  ${config_name} \
  $(jq -r '.rollupConfigHash, .aggregationVkey, .rangeVkeyCommitment' opsuccinctl2ooconfig.json) \
  --rpc-url $l1_rpc_url \
  --private-key $aggchain_manager_private_key
```

## Step 4: Select Configuration On-Chain

Activate the new configuration by calling `selectOpSuccinctConfig`.

**Function signature:**

```solidity
function selectOpSuccinctConfig(bytes32 _configName)
```

**Example using `cast`:**

```shell
cast send $rollup_address \
  "selectOpSuccinctConfig(bytes32)" \
  $config_name \
  --rpc-url $l1_rpc_url \
  --private-key $aggchain_manager_private_key
```
