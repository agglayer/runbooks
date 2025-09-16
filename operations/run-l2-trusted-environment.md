# Running L2 Trusted Environment

This document provides instructions for running a trusted environment of an L2 network. The procedure varies depending on the client stack being used.

## Overview

L2 trusted environments require different components and configurations based on the underlying stack:

- **CDK-Erigon Stack**: Networks using `cdk-erigon` as the execution client
- **Vanilla OP Stack**: Networks using standard OP Stack components (`op-geth`, `op-node`, `op-batcher`)

## Common Prerequisites

Before deploying any L2 trusted environment, ensure you have:

1. **Database**: PostgreSQL instance for bridge service
2. **L1 Access**: RPC endpoint to the L1 network (Ethereum Sepolia for testnet)
3. **Network Configuration**: Genesis file, chain specifications, and contract addresses
4. **Secrets Management**: Private keys for sequencer, batcher, and other services
5. **Persistent Storage**: For blockchain data and configuration files

## CDK-Erigon Stack

For networks using the CDK-Erigon stack, the following components must be deployed in this order:

### 1. PostgreSQL Database

**Purpose**: Database for bridge service

**Docker Image**: e.g., `docker.io/bitnami/postgresql`

**Startup Order**: Deploy first (dependency for other services)

**Configuration**: Standard PostgreSQL setup

### 2. Pool Manager

**Purpose**: Transaction pool management for CDK-Erigon

**Docker Image**: `docker.io/hermeznetwork/zkevm-pool-manager:v0.1.2`

**Startup Order**: Deploy after PostgreSQL

**Dependencies**: PostgreSQL, secrets

### 3. CDK-Erigon Sequencer

**Purpose**: Block production and sequencing

**Docker Image**: `docker.io/hermeznetwork/cdk-erigon:v2.61.23`

**Startup Order**: Deploy after secrets are available

**Configuration**:
- `is_sequencer: 1`
- Chain specification and genesis configuration
- L1 contract addresses

### 4. CDK-Erigon RPC Node

**Purpose**: JSON-RPC API server for L2 interactions

**Docker Image**: `docker.io/hermeznetwork/cdk-erigon:v2.61.23`

**Startup Order**: Deploy after pool-manager and cdk-erigon-sequencer

**Dependencies**: Pool Manager, CDK-Erigon Sequencer

**Configuration**:
- `is_sequencer: 0`
- RPC endpoints and API configuration

### 5. AggKit

**Purpose**: Agglayer integration and certificate management

**Docker Image**: `ghcr.io/agglayer/aggkit:0.5.1`

**Startup Order**: Deploy after CDK-Erigon RPC

**Dependencies**: CDK-Erigon RPC

**Components**:
- `aggsender`: Sends certificates to Agglayer
- `bridge`: Bridge service component

**Configuration**:

AggKit requires a TOML configuration file.

<details>
<summary>Configuration template</summary>

```toml
# Network ID of the L2 chain
NetworkID = 1

# Path to read/write data directory
PathRWData = "/data"

# L1 RPC URL for Ethereum connectivity (use your L1 provider)
L1URL = "https://your-l1-rpc-endpoint.com"

# L2 RPC URL pointing to your CDK-Erigon RPC service
L2URL = "http://cdk-erigon-rpc:8123"

# Path to sequencer private key file (leave empty to use other auth methods)
SequencerPrivateKeyPath = ""
# Password for sequencer private key (leave empty if not using file-based keys)
SequencerPrivateKeyPassword = ""

# Polygon bridge contract address on L2
polygonBridgeAddr = "0x0000000000000000000000000000000000000000"

# RPC URL for general chain operations (typically same as L2URL)
RPCURL = "http://cdk-erigon-rpc:8123"

# Witness service URL (optional, leave empty if not using)
WitnessURL = ""

# Block number where the rollup was created on L1
rollupCreationBlockNumber = 0
# Block number where the rollup manager was deployed on L1
rollupManagerCreationBlockNumber = 0
# Genesis block number on L1 (typically same as rollup manager creation)
genesisBlockNumber = 0

# L1 chain configuration
[L1Config]
# L1 chain ID (11155111 for Sepolia, 1 for Mainnet)
chainId = 11155111
# Global exit root contract address on L1
polygonZkEVMGlobalExitRootAddress = "0x0000000000000000000000000000000000000000"
# Rollup manager contract address on L1
polygonRollupManagerAddress = "0x0000000000000000000000000000000000000000"
# POL token contract address on L1
polTokenAddress = "0x0000000000000000000000000000000000000000"
# Rollup contract address on L1
polygonZkEVMAddress = "0x0000000000000000000000000000000000000000"

# L2 chain configuration
[L2Config]
# Global exit root contract address on L2
GlobalExitRootAddr = "0x0000000000000000000000000000000000000000"

# Logging configuration
[Log]
# Environment setting (development, production)
Environment = "production"
# Log level (debug, info, warn, error)
Level = "info"
# Log output destinations
Outputs = ["stderr"]

# RPC server configuration
[RPC]
# Port for RPC server to listen on
Port = 5576

# Agglayer sender configuration
[AggSender]
# Private key configuration for aggsender service
AggsenderPrivateKey = {Path = "/etc/aggkit/sequencer.keystore", Password = "***"}

# Interval for trying to send certificates to agglayer
CertificateSendInterval = "1m"
# Interval for checking if certificates are settled
CheckSettledInterval = "5s"
# Maximum certificate size in bytes
MaxCertSize = 8388608
# Path to save certificate files for debugging
SaveCertificatesToFilesPath = "/tmp"
# Block finality requirement (LatestBlock, SafeBlock, FinalizedBlock)
BlockFinality = "FinalizedBlock"
# Proof mode (PessimisticProof, FEP)
Mode = "PessimisticProof"
# Require no gap between FEP blocks
RequireNoFEPBlockGap = true

# Agglayer client configuration
[AggSender.AgglayerClient]
# Agglayer service URL (use appropriate environment: dev, test, or production)
URL = "grpc-agglayer-dev.polygon.technology:443"
# Use TLS for secure connection
UseTLS = "true"

# Bridge L2 sync configuration
[BridgeL2Sync]
# Bridge contract address on L2 (should match polygonBridgeAddr for the CDK-Erigon stack)
BridgeAddr = "0x0000000000000000000000000000000000000000"

# L1 info tree sync configuration
[L1InfoTreeSync]
# Block finality requirement for L1 sync
BlockFinality = "FinalizedBlock"
# Number of blocks to sync in each chunk
SyncBlockChunkSize = 1000
# Initial block to start syncing from
InitialBlock = 0

# REST API configuration (bridge)
[REST]
# Port for the bridge REST API server
Port = "5577"
```
</details>
</br>

**Command to run AggKit**:

```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,bridge
```

**Available Components**:
- `aggsender`: Sends certificates to the Agglayer
- `bridge`: Bridge service component
- `aggoracle`: Oracle services (for OP Stack)

**Component Selection**:
- For CDK-Erigon: typically use `aggsender,bridge`
- For OP Stack: typically use `aggsender,aggoracle,bridge`

**Environment-specific URLs**:

Agglayer URL
- Bali: `grpc-agglayer-dev.polygon.technology:443`
- Cardona: `grpc-agglayer-test.polygon.technology:443`
- Mainnet: `grpc-agglayer.polygon.technology:443`

### 6. Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `docker.io/hermeznetwork/zkevm-bridge-service:v0.6.2-RC2`

**Startup Order**: Deploy after CDK-Erigon RPC

**Dependencies**: CDK-Erigon RPC

**Configuration**: L1 and L2 contract addresses, claim transaction management

### 7. Bridge UI (Optional)

**Purpose**: Web interface for bridge operations

**Docker Image**: `docker.io/hermeznetwork/zkevm-bridge-ui:multi-network`

**Startup Order**: Deploy after AggKit

**Dependencies**: AggKit

## Vanilla OP Stack

For networks using the standard OP Stack, the following components must be deployed:

### 1. PostgreSQL Database

**Purpose**: Database for bridge service

**Docker Image**: e.g., `docker.io/bitnami/postgresql`

**Startup Order**: Deploy first

### 2. OP-Geth (Execution Layer)

**Purpose**: Ethereum execution client (L2)

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101503.1`

**Startup Order**: Deploy after secrets

**Configuration**:
- Archive or full node type
- L2 chain ID and block time configuration
- JSON-RPC API endpoints

### 3. OP-Node (Consensus Layer)

**Purpose**: Optimism consensus client

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.12.0`

**Startup Order**: Deploy after OP-Geth

**Dependencies**: OP-Geth, secrets

**Configuration**:
- Rollup configuration JSON
- L1 and L2 chain parameters
- System configuration

### 4. OP-Batcher

**Purpose**: Batch submission to L1

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.11.5`

**Startup Order**: Deploy after OP-Geth

**Dependencies**: OP-Geth, secrets

### 5. AggKit (Agglayer Integration)

**Purpose**: Agglayer integration and oracle services

**Docker Image**: `ghcr.io/agglayer/aggkit:0.5.1`

**Startup Order**: Deploy after OP-Batcher and OP-Geth

**Dependencies**: OP-Batcher, OP-Geth

**Components**:
- `aggsender`: Certificate management
- `aggoracle`: Oracle services for OP Stack
- `bridge`: Bridge service for cross-chain asset transfers

**Configuration**:
- Agglayer client URL: `grpc-agglayer[-dev|-test|].polygon.technology:443`
- L2 RPC URL: op-geth URL
- See AggKit Configuration section above for detailed config file template

### 6. Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `hermeznetwork/zkevm-bridge-service:v0.6.2-RC2`

**Startup Order**: Deploy after PostgreSQL and OP-Geth

**Dependencies**: PostgreSQL, OP-Geth

### 7. Bridge UI (Optional)

**Purpose**: Web interface for bridge operations

**Docker Image**: `docker.io/hermeznetwork/zkevm-bridge-ui:multi-network`

**Startup Order**: Deploy after Bridge Service

**Dependencies**: Bridge Service

## Component Version Matrix

### CDK-Erigon Stack Versions

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `docker.io/bitnami/postgresql:16.2.1` |
| Pool Manager | `docker.io/hermeznetwork/zkevm-pool-manager:v0.1.2` |
| CDK-Erigon | `docker.io/hermeznetwork/cdk-erigon:v2.61.23` |
| AggKit | `ghcr.io/agglayer/aggkit:0.5.1` |
| Bridge | `docker.io/hermeznetwork/zkevm-bridge-service:v0.6.2-RC2` |
| Bridge UI | `docker.io/hermeznetwork/zkevm-bridge-ui:multi-network` |

### Vanilla OP Stack Versions

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `docker.io/bitnami/postgresql:16.2.1` |
| OP-Geth | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101503.1` |
| OP-Node | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.12.0` |
| OP-Batcher | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.11.5` |
| AggKit | `ghcr.io/agglayer/aggkit:0.5.1` |
| Bridge | `docker.io/hermeznetwork/zkevm-bridge-service:v0.6.2-RC2` |
| Bridge UI | `docker.io/hermeznetwork/zkevm-bridge-ui:multi-network` |

## Deployment Considerations

### Resource Requirements

**CDK-Erigon Components:**
- CDK-Erigon Sequencer/RPC: High CPU and memory requirements
- Pool Manager: Moderate resources
- Storage: 500Gi for blockchain data

**OP Stack Components:**
- OP-Geth: High CPU and memory (archive mode)
- OP-Node: Moderate resources
- OP-Batcher: Low to moderate resources

### Network Configuration

**Common L1 Configuration (Sepolia):**
- Chain ID: 11155111
- Block time: 12 seconds
- RPC endpoints for contract interaction

**L2 Configuration varies by network:**
- Chain ID: Unique per deployment
- Block time: 2 seconds (typical)
- Contract addresses from L1 deployment

### Security Considerations

1. **Private Key Management**: All services require secure key management
2. **Network Isolation**: Restrict inter-service communication
3. **Secret Storage**: Use secure secret management systems
4. **Access Control**: Limit admin access to critical components

### Monitoring and Logging

1. **Log Levels**: Use `info` for production, `debug` for development
2. **Metrics**: Monitor block production, transaction processing
3. **Health Checks**: Implement service health monitoring
4. **Alerting**: Set up alerts for service failures

## Common Issues and Troubleshooting

### CDK-Erigon Stack Issues

1. **Sequencer Sync Issues**: Check L1 connectivity and contract addresses
2. **Pool Manager Connection**: Verify database connectivity
3. **Certificate Submission**: Check Agglayer connectivity and credentials

### OP Stack Issues

1. **OP-Node Sync**: Verify rollup configuration and L1 sync
2. **Batcher Failures**: Check L1 gas prices and wallet funding
3. **Geth Sync Issues**: Monitor peer connections and storage

### General Issues

1. **Database Connectivity**: Verify PostgreSQL accessibility
2. **Storage Issues**: Monitor disk space for blockchain data
3. **Network Connectivity**: Check inter-service communication
4. **Secret Access**: Verify key management and access permissions

## Upgrading Components

When upgrading components:

1. **Backup**: Always backup blockchain data and configuration
2. **Test**: Verify compatibility between component versions
3. **Rolling Updates**: Update non-consensus components first
4. **Consensus Components**: Coordinate updates for sequencer/consensus layers
5. **Rollback Plan**: Maintain ability to rollback to previous versions

This runbook should be updated whenever component versions change or new deployment patterns are established.
