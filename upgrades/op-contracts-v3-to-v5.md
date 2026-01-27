# OP Contracts Upgrade: v3.0.0 → v5.0.0

This runbook outlines the procedure for upgrading Optimism-stack contracts from op-contracts/v3.0.0 to op-contracts/v5.0.0 on a generic network. This is a stepped upgrade that requires passing through v4.1.0 before finalizing v5.0.0.

## Impact

- This upgrade is a pre-requisite for the Jovian Hardfork but does not strictly activate it
- It should not affect customers or external node operators

## Prerequisites

### Tools

- `cast` (part of Foundry) for data encoding/decoding
- `make` and `node` for running the signing scripts
- **Safer CLI**: A specialized Safe CLI tool is required because standard UIs often disable delegatecall transactions, which are necessary for this upgrade
- **Hardware**: Ledger or compatible hardware wallet for signing

### Network Info

- Access to the target network's RPC URL and contract addresses

## 1. Upgrade Variables

Before executing, identify the following addresses for your target network. These will be used to generate the transaction payloads.

| Variable | Description |
|----------|-------------|
| `V4_UPGRADER` | Address of the v4.1.0 Upgrade Manager contract |
| `V5_UPGRADER` | Address of the v5.0.0 Upgrade Manager contract |
| `SC_PROXY` | Superchain Config Proxy address |
| `SC_ADMIN` | Superchain ProxyAdmin address |
| `SYS_PROXY` | System Config Proxy address |
| `L1_ADMIN` | OP Chain L1 ProxyAdmin address |
| `PRESTATE` | Current Fault Game Prestate (32-byte hex) |
| `PORTAL_PROXY` | OptimismPortal Proxy address |
| `PORTAL_IMPL` | New OptimismPortal implementation (e.g., disables native deposits) |

## 2. Prepare Transaction Data

You must generate three distinct hex payloads using `cast`.

### Payload A: Superchain Config Upgrade

Used to upgrade the Superchain Config contract.

```bash
# Signature: upgradeSuperchainConfig(address _proxy, address _admin)
cast calldata "upgradeSuperchainConfig(address,address)" <SC_PROXY> <SC_ADMIN>
```

Save this output as `DATA_A`.

### Payload B: OP Chain Batch Upgrade

Used to upgrade the System Config and other proxies. This bundles multiple upgrades into one call.

```bash
# Signature: upgrade((address _proxy, address _admin, bytes32 _prestate)[])
# Note: Ensure the tuple array is enclosed in brackets [ ... ]
cast calldata "upgrade((address,address,bytes32)[])" "[(<SYS_PROXY>, <L1_ADMIN>, <PRESTATE>)]"
```

Save this output as `DATA_B`.

### Payload C: Portal Implementation Swap

Used to point the OptimismPortal proxy to the new custom implementation.

```bash
# Signature: upgrade(address _proxy, address _impl)
cast calldata "upgrade(address,address)" <PORTAL_PROXY> <PORTAL_IMPL>
```

Save this output as `DATA_C`.

## 3. Construct Transaction Bundle

Create a file named `data/batch.json` in your Safer CLI directory. The bundle consists of 5 transactions to ensure the stepped upgrade path (v3 → v4 → v5).

**Note on Operations:**
- `operation: 1` = DelegateCall (Required for Upgrader contracts)
- `operation: 0` = Call (Standard interaction)

```json
[
  {
    "comment": "1. Upgrade Superchain contracts to v4.1.0",
    "to": "<V4_UPGRADER>",
    "operation": 1,
    "value": "0",
    "data": "<DATA_A>"
  },
  {
    "comment": "2. Upgrade Superchain contracts to v5.0.0",
    "to": "<V5_UPGRADER>",
    "operation": 1,
    "value": "0",
    "data": "<DATA_A>"
  },
  {
    "comment": "3. Upgrade OP Chain contracts to v4.1.0",
    "to": "<V4_UPGRADER>",
    "operation": 1,
    "value": "0",
    "data": "<DATA_B>"
  },
  {
    "comment": "4. Upgrade OP Chain contracts to v5.0.0",
    "to": "<V5_UPGRADER>",
    "operation": 1,
    "value": "0",
    "data": "<DATA_B>"
  },
  {
    "comment": "5. Point Optimism Portal to custom impl (disable native deposits)",
    "to": "<L1_ADMIN>",
    "operation": 0,
    "value": "0",
    "data": "<DATA_C>"
  }
]
```

## 4. Validation

Before signing, verify the batch contents:

1. **Decode Data**: Use `cast 4bd <DATA>` to verify that the hex strings resolve to the expected function names (`upgradeSuperchainConfig`, `upgrade`) and addresses
2. **Simulation**: Use Tenderly or a similar tool to simulate the transaction bundle. Ensure the simulation succeeds and state changes are reflected correctly

## 5. Execution (Signing)

### Setup Environment

Create a `.env` file in the root of the Safer repository:

```bash
SAFE=<YOUR_SAFE_ADDRESS>
SENDER=<YOUR_WALLET_ADDRESS>
MULTI_SEND_ADDR=<NETWORK_MULTISEND_ADDR>
SAFE_NONCE=<CURRENT_SAFE_NONCE>
FOUNDRY_ETH_RPC_URL=<NETWORK_RPC_URL>
```

### Process

1. **Prepare Batch**: Run the preparation script to parse the JSON

```bash
make prepare-batch
```

2. **Verify Hash**: Generate the transaction hash. All signers must verify this hash matches exactly

```bash
make hash
# Compare output in data/hashData.txt with other signers
```

3. **Sign**: Connect your Ledger and sign the hash

```bash
make sign:ledger
```

4. **Broadcast**: Once the required threshold of signatures (e.g., 3/5) is collected in `data/signatures.txt`, execute the batch. (Ping the designated executor to run the final make command)
