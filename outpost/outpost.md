# Outpost Configuration Reference

## Data Sources

| Data | File |
|------|------|
| L1 info | `combined.json` |
| L2 info | `outpost.json` |

## AggKit Configuration

| Config | Value | Reference |
|--------|-------|-----------|
| `AggOracle.EVMSender.EthTxManager.Etherman.L1ChainID` | Must match the **outpost chain**, not the rollup chain ID | [default.go#L137](https://github.com/agglayer/aggkit/blob/develop/config/default.go#L137) |

### Aggsender

#### Proposer

See [Proposer](../operations/run-aggsender-committee.md#scenario-1-proposer-own-l2-chain).

| Config | Value | Reference |
|--------|-------|-----------|
| `AggSender.ValidatorClient.UseTLS` | `true` | [default.go#L267](https://github.com/agglayer/aggkit/blob/develop/config/default.go#L267) |

#### Validator

See [Validator](../operations/run-aggsender-committee.md#scenario-2-validator-committee-participation).

| Config | Value | Reference |
|--------|-------|-----------|
| `Validator.EnableRPC` | `true` (must be publicly reachable) | [default.go#L297](https://github.com/agglayer/aggkit/blob/develop/config/default.go#L297) |

### Aggoracle

See [AggOracle Committee](../operations/run-aggoracle-committee.md#mode-2-aggoracle-committee-decentralized).

| Config | Value | Reference |
|--------|-------|-----------|
| `L2Config.AggOracleCommitteeAddr` | Configure the committee address | [default.go#L34](https://github.com/agglayer/aggkit/blob/develop/config/default.go#L34) |
| `AggOracle.EnableAggOracleCommittee` | `true` | [default.go#L110](https://github.com/agglayer/aggkit/blob/develop/config/default.go#L110) |
