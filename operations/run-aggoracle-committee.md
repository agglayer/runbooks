# Run AggOracle Committee

This runbook provides guidance for Implementation Providers (IPs) on deploying and configuring the `aggoracle` component and managing the AggOracle Committee. The `aggoracle` is responsible for reading the Global Exit Root (GER) from L1 and proposing or injecting it into L2, either directly or through a decentralized committee.

## Overview

The `aggoracle` component serves two primary deployment modes for Implementation Providers:

1. **Direct Injection Mode**: The oracle directly injects GERs into the L2 Global Exit Root Manager contract
2. **AggOracle Committee Mode**: The oracle participates in a decentralized committee that proposes GERs, requiring quorum consensus before injection

The AggOracle Committee provides enhanced security through decentralized consensus while maintaining the ability to update Global Exit Roots on L2.

## Deployment Modes

### Mode 1: Direct Injection (Traditional)

**When to use**: You want a simple, direct oracle setup where the aggoracle directly injects GERs into the L2 Global Exit Root Manager.

**Components to run**:
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggoracle
```

**Key characteristics**:
- Single oracle directly injects GERs
- Lower latency for GER updates
- Requires the oracle address to be set as the `globalExitRootUpdater` in the L2 Global Exit Root Manager
- Suitable for trusted environments or single-operator chains

### Mode 2: AggOracle Committee (Decentralized)

**When to use**: You want enhanced security through decentralized consensus for GER updates.

**Components to run**:
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggoracle
```

**Key characteristics**:
- Multiple oracle members participate in committee
- Requires quorum consensus before GER injection
- Enhanced security through decentralization
- Committee members propose GERs, which are automatically injected when quorum is reached
- Suitable for production environments with multiple trusted operators

## Configuration Requirements

### Core Configuration Sections

All `aggoracle` deployments require configuration of the oracle-specific section. The configuration follows the standard AggKit format and integrates with the global configuration variables.

<details>
<summary>AggOracle configuration section</summary>

```toml
# Global configuration variables (used throughout config)
L1URL = "https://eth-mainnet.g.alchemy.com/v2/YOUR-API-KEY"
L2URL = "https://your-l2-rpc-endpoint"

# L2 contract addresses
[L2Config]
    # Address of the sovereign global exit root proxy contract on L2
    GlobalExitRootAddr = "0x0000000000000000000000000000000000000000"
    # Address of the aggoracle committee (required for committee mode)
    AggOracleCommitteeAddr = "0x0000000000000000000000000000000000000000"

# AggOracle configuration
[AggOracle]
    # Enable AggOracle Committee mode (false for direct injection)
    EnableAggOracleCommittee = false

    [AggOracle.EVMSender]
        # L2 Global Exit Root Manager contract address (references global config)
        GlobalExitRootL2 = "{{L2Config.GlobalExitRootAddr}}"

        # AggOracle Committee contract address (references global config)
        AggOracleCommitteeAddr = "{{L2Config.AggOracleCommitteeAddr}}"

        [AggOracle.EVMSender.EthTxManager]
            # Oracle private key configuration
            PrivateKeys = [
                {Method = "local", Path = "/app/keystore/aggoracle.keystore", Password = "testonly"}
            ]

            [AggOracle.EVMSender.EthTxManager.Etherman]
                # L1 Chain ID
                L1ChainID = 1
```

</details>

### Mode-Specific Configuration

#### Direct Injection Mode

For direct injection mode, set `EnableAggOracleCommittee = false` and ensure:

1. The oracle's private key corresponds to the address that is set as the `globalExitRootUpdater` in the L2 Global Exit Root Manager contract
2. The `AggOracleCommitteeAddr` field should _not_ be set in the config file for direct injection mode

#### AggOracle Committee Mode

For committee mode, set `EnableAggOracleCommittee = true` and ensure:

1. The oracle's private key corresponds to an address that is a member of the AggOracle Committee
2. The `AggOracleCommitteeAddr` is set to the deployed AggOracle Committee contract address

## AggOracle Committee Management

### Committee Structure

The AggOracle Committee operates on a multi-signature consensus model:

- **Committee Members**: Array of oracle addresses that can propose GERs
- **Quorum**: Minimum number of votes required for GER consolidation
- **Proposal Process**: Members propose GERs, and when quorum is reached, the GER is automatically injected
- **Owner**: Contract owner who can manage committee members and quorum

### Committee Configuration Parameters

When creating the genesis with an AggOracle Committee, the following parameters must be configured:

```json
{
  "useAggOracleCommittee": true,
  "aggOracleCommittee": [
    "0x...",
    "0x...",
    "0x..."
  ],
  "quorum": 2
}
```

**Parameter Requirements**:
- `useAggOracleCommittee` must be set to `true` to enable committee mode
- `quorum` must be ≥ 1 and ≤ number of committee members
- All `aggOracleCommittee` addresses must be valid and unique
- Additional parameters like `aggOracleOwner` will be required for full genesis configuration

### Managing Committee Members

#### Adding Oracle Members

The committee owner can add new oracle members using the `addOracleMember` function:

```bash
cast send <AGGORACLE_COMMITTEE_ADDRESS> \
  "addOracleMember(address)" \
  <NEW_ORACLE_ADDRESS> \
  --rpc-url <L2_RPC_URL> \
  --private-key <OWNER_PRIVATE_KEY>
```

#### Removing Oracle Members

To remove an oracle member, you need both the address and their index in the array:

```bash
# First, get the member's index
cast call <AGGORACLE_COMMITTEE_ADDRESS> \
  "getAggOracleMemberIndex(address)" \
  <ORACLE_ADDRESS> \
  --rpc-url <L2_RPC_URL>

# Then remove the member
cast send <AGGORACLE_COMMITTEE_ADDRESS> \
  "removeOracleMember(address,uint256)" \
  <ORACLE_ADDRESS> <MEMBER_INDEX> \
  --rpc-url <L2_RPC_URL> \
  --private-key <OWNER_PRIVATE_KEY>
```

#### Updating Quorum

The committee owner can update the quorum requirement:

```bash
cast send <AGGORACLE_COMMITTEE_ADDRESS> \
  "updateQuorum(uint64)" \
  <NEW_QUORUM> \
  --rpc-url <L2_RPC_URL> \
  --private-key <OWNER_PRIVATE_KEY>
```

### Querying Committee Information

#### Get All Committee Members

```bash
cast call <AGGORACLE_COMMITTEE_ADDRESS> \
  "getAllAggOracleMembers()" \
  --rpc-url <L2_RPC_URL>
```

#### Get Current Quorum

```bash
cast call <AGGORACLE_COMMITTEE_ADDRESS> \
  "quorum()" \
  --rpc-url <L2_RPC_URL>
```

## Key Management

### Private Key Requirements

**Direct Injection Mode**:
- Oracle private key (must be set as `globalExitRootUpdater`)

**AggOracle Committee Mode**:
- aggOracle private key (must be a committee member)
- aggOracleOwner private key (for administrative functions)

## References

For full reference and additional configuration options, check out the repository: https://github.com/agglayer/aggkit
