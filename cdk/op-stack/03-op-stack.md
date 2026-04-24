# Network Deployment (OP Stack)

This document explains how to deploy an L2 using the OP Stack's `op-deployer` tool. It follows the official Optimism deployment flow and focuses on the steps most relevant to this repository.

## References

- [Optimism L2 Rollup Tutorial](https://docs.optimism.io/operators/chain-operators/tutorials/create-l2-rollup)
- [op-deployer Tool Documentation](https://docs.optimism.io/operators/chain-operators/tools/op-deployer)

> **Note**: All commands assume you're running from the repository root.

## TL;DR

1. Initialize an `op-deployer` workdir
2. Edit `intent.toml` with deployment parameters
3. Deploy L1 contracts using `op-deployer apply`
4. Merge existing genesis allocs (e.g., Polygon) into the op-deployer state
5. Generate final `genesis.json` and `rollup.json` files via `op-deployer inspect`

### Environment Variables

Set the following environment variables before proceeding:

```shell
# 11155111 for Sepolia
export l1_chain_id=<your-l1-chain_id>
export l2_chain_id=<l2ChainID-from-combined.json>
export l1_rpc_url="https://<your_l1_rpc>"
export l1_rpc_url_wss="wss://<your_l1_rpc>"
export deployer_private_key=0x... # private key of DEPLOYER_ADDRESS
# See Component Versions table in README.md for current values
export op_deployer_version="<op_deployer_version>"
export op_reth_version="<op_reth_version>"
```

## Step 1: Initialize the Deployer Workdir

Create a local `deployer` folder and initialize the op-deployer state:

```shell
docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	init \
		--l1-chain-id ${l1_chain_id} \
		--l2-chain-ids ${l2_chain_id} \
		--workdir /deployer
```

Edit the generated `deployer/intent.toml` with your deployment parameters. You will particularly need to generate new addresses for:

- `unsafeBlockSigner` (SEQUENCER_ADDRESS)
- `batcher` (BATCHER_ADDRESS)

Here's an `intent.toml` example file with the right modifications:

```toml
configType = "standard-overrides"
opDeployerVersion = "<generated>"
l1ChainID = <your-l1-chain_id>
opcmAddress = "<generated>"
fundDevAccounts = true
l1ContractsLocator = "embedded"
l2ContractsLocator = "embedded"

[globalDeployOverrides]
  l2BlockTime = 1

[[chains]]
  id = "<generated>"
  baseFeeVaultRecipient = "<l2ChainID-from-combined.json>"
  l1FeeVaultRecipient = "<ADMIN_ADDR>"
  sequencerFeeVaultRecipient = "<ADMIN_ADDR>"
  operatorFeeVaultRecipient = "<ADMIN_ADDR>"
  eip1559DenominatorCanyon = 250
  eip1559Denominator = 50
  eip1559Elasticity = 6
  gasLimit = 60000000
  operatorFeeScalar = 0
  operatorFeeConstant = 0
  useRevenueShare = true
  chainFeesRecipient = "<ADMIN_ADDR>"
  minBaseFee = 0
  daFootprintGasScalar = 0
  [chains.roles]
    l1ProxyAdminOwner = "<ADMIN_ADDR>"
    l2ProxyAdminOwner = "<ADMIN_ADDR>"
    systemConfigOwner = "<ADMIN_ADDR>"
    unsafeBlockSigner = "<SEQUENCER_ADDRESS>"
    batcher = "<BATCHER_ADDRESS>"
    proposer = "<ADMIN_ADDR>"
    challenger = "<ADMIN_ADDR>"


```

## Step 2: Deploy L1 Contracts

When your `intent.toml` is ready, deploy the L1 contracts required by the OP Stack:

```shell
docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	apply \
		--workdir /deployer \
		--l1-rpc-url ${l1_rpc_url} \
		--private-key ${deployer_private_key}
```

This writes the deployer state to `deployer/state.json` and related artifacts.

## Step 3: Merge OP + Polygon Genesis with Pre-deployed Contracts

This step is required when you have pre-deployed contracts or existing chain allocs (for example, from a `polygon-genesis.json`). You must merge those allocs into the op-deployer state so the final L2 genesis includes the pre-deployed addresses and balances.

The commands below:
1. Extract the base64/gzip-encoded allocs from the op-deployer state
2. Merge them with your Polygon alloc fragment
3. Write the merged allocs back into the state

```shell
# Extract the allocs
cat deployer/state.json | jq -r '.opChainDeployments[].allocs' | base64 -d | gzip -d > allocs.json

# Merge with your Polygon genesis file
jq -s add allocs.json <path/to/polygon-genesis.json> | gzip | base64 > merge

# Create a copy of the original state
cp deployer/state.json deployer/original-state.json

# Replace the original allocs by the merged
cat deployer/state.json | jq ".opChainDeployments[].allocs=\"$( cat merge )\"" > state.json && mv state.json deployer/state.json

# Cleanup
rm allocs.json merge
```

## Step 4: Generate Final Artifacts

Once the state is ready, use `op-deployer inspect` to produce the final `genesis.json` and `rollup.json` for the L2:

```shell
docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	inspect genesis --workdir /deployer ${l2_chain_id} > ./deployer/genesis.json

docker run --rm -v "$(pwd)/deployer:/deployer" --entrypoint /usr/local/bin/op-deployer \
	us-docker.pkg.dev/oplabs-tools-artifacts/images/op-deployer:${op_deployer_version} \
	inspect rollup --workdir /deployer ${l2_chain_id} > ./deployer/rollup.json
```

These files serve as inputs for running the OP Stack nodes or for further tooling in this repository.

## OP Stack Components

After deploying the contracts and generating the genesis files, you'll need to run the OP Stack components. For complete configuration documentation, refer to the official Optimism documentation:

### Sequencer

> **Source of Truth**: [Spinning up the sequencer](https://docs.optimism.io/chain-operators/guides/deployment/sequencer-node)

The sequencer consists of two core components:
- **op-reth**: Execution layer that processes transactions and maintains state
- **op-node**: Consensus layer that orders transactions and creates L2 blocks

The sequencer is responsible for ordering transactions from users, building L2 blocks, and signing blocks on the P2P network.

### Batcher

> **Source of Truth**: [Spinning up the batcher](https://docs.optimism.io/chain-operators/guides/deployment/spin-batcher)

The batcher (`op-batcher`) collects L2 transactions and submits them as batches to L1. It ensures L2 transaction data is available on L1 for data availability and enables users to reconstruct the L2 state.

