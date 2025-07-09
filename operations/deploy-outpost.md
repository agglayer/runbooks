[TOC]

# Outposts Deployment Guideline

|             Title             |  Author   | Status | Created At |
|:-----------------------------:|:---------:|:------:|:----------:|
| Outpost Deployment Guidelines | krlosMata | Draft  | 09-07-2025 |

---

# Motivation
This document aims to provide a clear path for deploying the AggLayer smart contracts on an external network that is not native to the AggLayer.

A **non-native AggLayer chain** refers to one that has its own native bridge and finality mechanism—also called an **Outpost chain**.

This guideline includes:

- Step-by-step procedure for the deployment process 
- Security analysis and recommendations on parameters used
- Clear documentation and data required for proper auditing

---

# Assumptions
This document assumes the chain connected to AggLayer uses the [Pessimistic Consensus SC](https://github.com/agglayer/agglayer-contracts/blob/v11.0.0-rc.2/contracts/v2/consensus/pessimistic/PolygonPessimisticConsensus.sol), and is therefore secured only by the PP proof.

Using a different [consensus SC](https://github.com/agglayer/agglayer-contracts/tree/v11.0.0-rc.2/contracts/v2/consensus) may require additional parameter configuration.

---

# Deployment Process
## Summary Table

|            Step             |           Responsibility           |   Outcome    |
|:---------------------------:|:----------------------------------:|:------------:|
|     attachAggchainToAL      |              Polygon               |   rollupID   |
| Deploy Sovereign Chains SCs | Outpost chain - audited by Polygon | SC addresses |

:::info
The deployment process is **sequential**—each step must be completed in order.

The first step, `attachAggchainToAL`, will output a [rollupID](https://github.com/agglayer/agglayer-contracts/blob/v11.0.0-rc.2/contracts/v2/PolygonRollupManager.sol#L632) that uniquely identifies the chain.
[This rollupID is then used to deploy the Sovereign SCs](https://github.com/agglayer/agglayer-contracts/blob/v11.0.0-rc.2/contracts/v2/sovereignChains/BridgeL2SovereignChain.sol#L187) on the outpost chain.
:::

---

## attachAggchainToAL
### Summary Table

|                Parameter                |                      Brief Description                       | Security Risks |
|:---------------------------------------:|:------------------------------------------------------------:|:--------------:|
|              rollupTypeID               |     Determines SC implementation and verifier to be used     |    🔴 High     |
|                 chainID                 |       Metadata. Cannot be reused with other AggChains        |   🟠 Medium    |
|      initializeBytesAggchain:admin      | Manages the `trustedSequencer` address (aggsender component) |    🔴 High     |
|    initializeBytesAggchain:sequencer    |               Aggsender address configuration                |    🔴 High     |
| initializeBytesAggchain:gasTokenAddress |            Metadata. Not linked with L2 gas token            |     🟢 Low     |
|  initializeBytesAggchain:sequencerURL   |        Metadata. For decentralized RPC node discovery        |     🟢 Low     |
|   initializeBytesAggchain:networkName   |            Metadata. Network identifier on L1 SC             |     🟢 Low     |

### Detailed Parameter Explanation
#### `rollupTypeID`
- **Responsibility**: Polygon
- **Security Assumption**: 🔴 Incorrect type could lead to unexpected L1 behavior.

#### `chainID`
- **Responsibility**: Chain
- **Security Assumption**: 🟠 Metadata; must be consistent to avoid confusion.

#### `initializeBytesAggchain:admin`
- **Responsibility**: IP
- **Security Assumption**: 🔴 Controls the `trustedSequencer`. If compromised, the admin can change the sequencer.
- **Recommended Account**: Multisig

#### `initializeBytesAggchain:sequencer`
- **Responsibility**: IP
- **Security Assumption**: 🔴 The Aggsender address. It can submit certificates and advance PP state.
- **Recommended Account**: EOA

#### `initializeBytesAggchain:gasTokenAddress`
- **Responsibility**: Chain
- **Security Assumption**: 🟢 Metadata; consistency required.

#### `initializeBytesAggchain:sequencerURL`
- **Responsibility**: Chain
- **Security Assumption**: 🟢 Metadata; consistency required.

#### `initializeBytesAggchain:networkName`
- **Responsibility**: Chain
- **Security Assumption**: 🟢 Metadata; consistency required.

---

## Deploy Sovereign Chains SCs
Two smart contracts must be deployed:

- [`BridgeL2SovereignChain`](https://github.com/agglayer/agglayer-contracts/blob/v11.0.0-rc.2/contracts/v2/sovereignChains/BridgeL2SovereignChain.sol)
- [`GlobalExitRootManagerL2SovereignChain`](https://github.com/agglayer/agglayer-contracts/blob/v11.0.0-rc.2/contracts/v2/sovereignChains/GlobalExitRootManagerL2SovereignChain.sol)

### ⚠️ SC Deployment Guidelines
- **Deployment Requirements**:
  - MUST use [TransparentProxy pattern](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol)
  - Proxy admin MUST be an [AdminProxy SC](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/ProxyAdmin.sol)
  - The `ProxyAdmin` owner MUST be a [Timelock SC](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/TimelockController.sol)

- **Security Considerations**:
  - Malicious upgrades can result in permanent loss of funds.
  - Timelock delay should be at least **3 days** (to be agreed among Polygon, Chain, and IP).
  - Timelock proposer/executor roles should be controlled by a **multisig** (to be agreed among Polygon, Chain, and IP).

- **Circular Dependency**:
  - `BridgeL2SovereignChain` needs `GlobalExitRootManagerL2SovereignChain` during initialization.
  - `GlobalExitRootManagerL2SovereignChain` constructor needs the bridge address.
  - Two valid approaches:
    - **Sequential**:
      - Deploy `BridgeL2SovereignChain` without initializing.
      - Deploy `GlobalExitRootManagerL2SovereignChain`.
      - Initialize `BridgeL2SovereignChain`.
    - **Precalculate addresses**:
      - Precompute contract addresses.
      - Deploy with proper references.

---

### `BridgeL2SovereignChain`: Summary Table
|              Parameter              |               Description                | Security Risks |
|:-----------------------------------:|:----------------------------------------:|:--------------:|
|             `networkID`             |   RollupID from `PolygonRollupManager`   |    🔴 High     |
|          `gasTokenAddress`          |    Address of gas token on the chain     |    🔴 High     |
|          `gasTokenNetwork`          |        Origin chain of gas token         |    🔴 High     |
|       `globalExitRootManager`       |       Address of associated GER SC       |    🔴 High     |
|       `polygonRollupManager`        |                 Not used                 |     🟢 Low     |
|         `gasTokenMetadata`          | Metadata of token (see “Setup GasToken”) |    🔴 High     |
|           `bridgeManager`           |          Handles token mappings          |    🔴 High     |
|       `sovereignWETHAddress`        |    WETH handler (must be `0x00...00`)    |    🔴 High     |
| `sovereignWETHAddressIsNotMintable` |             Must be `false`              |    🔴 High     |
|       `emergencyBridgePauser`       |           Can pause the bridge           |    🔴 High     |
|      `emergencyBridgeUnPauser`      |          Can unpause the bridge          |   🟠 Medium    |
|       `proxiedTokensManager`        |    Owner of all wrapped token proxies    |    🔴 High     |

#### Setup GasToken
Three key parameters:
- `gasTokenAddress`
- `gasTokenNetwork`
- `gasTokenMetadata` (`abi.encode("Name", "Symbol", decimals)`)

Two scenarios:
- **External token**:
  - `gasTokenAddress`: address on origin chain
  - `gasTokenNetwork`: origin chain ID
  - `gasTokenMetadata`: encoded metadata
- **Native L2 token**:
  - `gasTokenAddress`: derived from rollupID — repeat rollupID (32 bits) 5x → 160-bit address. i.e: `0x{rollupID}{rollupID}{rollupID}{rollupID}{rollupID}`
  - `gasTokenNetwork`: chain rollupID
  - `gasTokenMetadata`: metadata shown when bridged

---

### `GlobalExitRootManagerL2SovereignChain`: Summary Table
|          Parameter          |             Description             | Security Risks |
|:---------------------------:|:-----------------------------------:|:--------------:|
| `constructor:bridgeAddress` | Address of `BridgeL2SovereignChain` |    🔴 High     |
|   `globalExitRootUpdater`   |        Address of AggOracle         |    🔴 High     |
|   `globalExitRootRemover`   |  Can remove GERs and unset claims   |    🔴 High     |

---

### `GlobalExitRootManagerL2SovereignChain`: Detailed Parameter Explanation
#### `constructor:bridgeAddress`
- **Responsibility**: Chain
- **Security**: 🔴 Blocking the LER update process causes DoS

#### `globalExitRootUpdater`
- **Responsibility**: Chain
- **Security**: 🔴 Can submit malicious GERs
- **Recommended Account**: EOA with additional protections

#### `globalExitRootRemover`
- **Responsibility**: Chain
- **Security**: 🔴 Can allow double-spending
- **Recommended Account**: Multisig

---

# Audit Requirements
- Use a GitHub issue (or similar) to track the audit process (**Slack threads are not acceptable**)
  - Example format: [Polygon audit issues](https://github.com/orgs/0xPolygon/projects/15)
- Useful data to make the audit smooth
  - documentation
  - block-explorer
  - RPC 
- Audit Scope
  - Tag/Commit on the script to deploy the SC
  - All addresses and parameters outlined in this document