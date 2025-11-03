# Running L2 Trusted Environment

This document provides instructions for running a trusted environment of an L2 network. The procedure varies depending on the client stack being used.

## Overview

L2 trusted environments require different components and configurations based on the underlying stack:

- **AggchainECDSAMultisig**: Networks using ECDSA multisig verification
  - **CDK-Erigon**: Networks using `cdk-erigon` as the execution client
  - **OP Stack**: Networks using standard OP Stack components (`op-geth`, `op-node`, `op-batcher`)
- **AggchainFEP**: Networks using Full Execution Proof (FEP)
  - **OP Stack**: Networks using standard OP Stack components (`op-geth`, `op-node`, `op-batcher`) and proving system (`op-succinct-proposer`, `aggkit-prover`)

## Common Prerequisites

Before deploying any L2 trusted environment, ensure you have:

1. **Database**: PostgreSQL instance for legacy bridge service
2. **L1 Access**: RPC endpoint to the L1 network (Ethereum Sepolia for testnet)
3. **Network Configuration**: Genesis file, chain specifications, and contract addresses
4. **Secrets Management**: Private keys for sequencer, batcher, and other services
5. **Persistent Storage**: For blockchain data and configuration files

## AggchainECDSAMultisig
### CDK-Erigon
#### 1. PostgreSQL Database

**Purpose**: Database for legacy bridge service

**Docker Image**: e.g., `bitnamisecure/postgresql@sha256:05f12b9dc62012ac6987bf3160241d2cbdeb60cf6d245f772d8582f89371929f`

**Startup Order**: Deploy first (dependency for other services)

**Configuration**: Standard PostgreSQL setup

#### 2. Pool Manager

**Purpose**: Transaction pool management for CDK-Erigon

**Docker Image**: `ghcr.io/0xpolygon/zkevm-pool-manager:v0.1.2`

**Startup Order**: Deploy after PostgreSQL

**Dependencies**: PostgreSQL, secrets

#### 3. CDK-Erigon Sequencer

**Purpose**: Block production and sequencing

**Docker Image**: `ghcr.io/0xpolygon/cdk-erigon:v2.61.23`

**Startup Order**: Deploy after secrets are available

**Configuration**:
- `is_sequencer: 1`
- Chain specification and genesis configuration
- L1 contract addresses

#### 4. CDK-Erigon RPC Node

**Purpose**: JSON-RPC API server for L2 interactions

**Docker Image**: `ghcr.io/0xpolygon/cdk-erigon:v2.61.23`

**Startup Order**: Deploy after pool-manager and cdk-erigon-sequencer

**Dependencies**: Pool Manager, CDK-Erigon Sequencer

**Configuration**:
- `is_sequencer: 0`
- RPC endpoints and API configuration

#### 5. Aggkit (Primary Instance)

**Purpose**: Agglayer integration and certificate management

**Docker Image**: `ghcr.io/agglayer/aggkit:0.7.0`

**Startup Order**: Deploy after CDK-Erigon RPC

**Dependencies**: CDK-Erigon RPC

**Components**:
- `aggsender`: Sends certificates to Agglayer

> [!IMPORTANT]
> **Two-Instance Architecture**
>
> Starting with Aggkit v0.7.0, IPs should run **two separate Aggkit instances**:
> 1. **Primary Instance**: Runs `aggsender` only - handles critical certificate submission
> 2. **Bridge Instance**: Runs `bridge` only - provides public REST API
>
> This separation ensures that public bridge API traffic does not impact critical certificate submission operations. See the [Run Aggkit Bridge](./run-aggkit-bridge.md) runbook for bridge deployment details.

**Configuration**:

Aggkit requires a TOML configuration file.

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

# Prometheus metrics configuration
[Prometheus]
# Enable Prometheus metrics endpoint (true to expose metrics, false to disable)
Enabled = true
# Host address for Prometheus metrics server to listen on (0.0.0.0 for all interfaces)
Host = "0.0.0.0"
# Port for Prometheus metrics server to listen on
Port = 9091

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

# L1 info tree sync configuration
[L1InfoTreeSync]
# Block finality requirement for L1 sync
BlockFinality = "FinalizedBlock"
# Number of blocks to sync in each chunk
SyncBlockChunkSize = 1000
```
</details>
</br>

**Command to run Aggkit (Primary Instance)**:

```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender
```

**Environment-specific URLs**:

Agglayer URL
- Bali: `grpc-agglayer-dev.polygon.technology:443`
- Cardona: `grpc-agglayer-test.polygon.technology:443`
- Mainnet: `grpc-agglayer.polygon.technology:443`

**Monitoring Aggkit with Prometheus**:

Aggkit exposes Prometheus metrics for monitoring the health and performance of the aggsender and other components. To enable metrics collection:

1. Configure the `[Prometheus]` section in your Aggkit config file (see configuration template above)
2. Set `Enabled = true` to expose the metrics endpoint
3. Configure your Prometheus server to scrape metrics from `http://<aggkit-host>:<prometheus-port>/metrics` (default port: 9091)

#### 6. Aggkit Bridge (Separate Instance)

Deploy the Aggkit bridge as a separate instance to provide the public bridge REST API. See the [Run Aggkit Bridge](./run-aggkit-bridge.md) runbook for complete deployment instructions.

**Key Points**:
- Run bridge as a separate instance from the primary Aggkit
- Bridge endpoint must be publicly accessible for Polygon's Bridge Hub

#### 7. Legacy Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `ghcr.io/0xpolygon/zkevm-bridge-service:v0.6.2`

**Startup Order**: Deploy after CDK-Erigon RPC

**Dependencies**: CDK-Erigon RPC

**Configuration**: L1 and L2 contract addresses, claim transaction management

#### 8. Legacy Bridge UI (Optional)

**Purpose**: Web interface for legacy bridge operations

**Docker Image**: `ghcr.io/0xpolygon/zkevm-bridge-ui:multi-network`

**Startup Order**: Deploy after Aggkit

**Dependencies**: Aggkit

### OP Stack
#### 1. PostgreSQL Database

**Purpose**: Database for legacy bridge service

**Docker Image**: e.g., `bitnamisecure/postgresql@sha256:05f12b9dc62012ac6987bf3160241d2cbdeb60cf6d245f772d8582f89371929f`

**Startup Order**: Deploy first (dependency for other services)

#### 2. OP-Geth (Execution Layer)

**Purpose**: Ethereum execution client (L2)

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101602.0`

**Startup Order**: Deploy after secrets

**Configuration**:
- Archive or full node type
- L2 chain ID and block time configuration
- JSON-RPC API endpoints

#### 3. OP-Node (Consensus Layer)

**Purpose**: Optimism consensus client

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.13.2`

**Startup Order**: Deploy after OP-Geth

**Dependencies**: OP-Geth, secrets

**Configuration**:
- Rollup configuration JSON
- L1 and L2 chain parameters
- System configuration

#### 4. OP-Batcher

**Purpose**: Batch submission to L1

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.11.5`

**Startup Order**: Deploy after OP-Geth

**Dependencies**: OP-Geth, secrets

#### 5. Aggkit (Primary Instance)

**Purpose**: Agglayer integration and oracle services

**Docker Image**: `ghcr.io/agglayer/aggkit:0.7.0`

**Startup Order**: Deploy after OP-Batcher and OP-Geth

**Dependencies**: OP-Batcher, OP-Geth

**Components**:
- `aggsender`: Certificate management
- `aggoracle`: Oracle services for OP Stack

> [!IMPORTANT]
> **Two-Instance Architecture**
>
> Starting with Aggkit v0.7.0, IPs should run **two separate Aggkit instances**:
> 1. **Primary Instance**: Runs `aggsender,aggoracle` - handles critical certificate submission and oracle services
> 2. **Bridge Instance**: Runs `bridge` only - provides public REST API
>
> This separation ensures that public bridge API traffic does not impact critical certificate submission operations. See the [Run Aggkit Bridge](./run-aggkit-bridge.md) runbook for bridge deployment details.

**Configuration**:
- Agglayer client URL: `grpc-agglayer[-dev|-test|].polygon.technology:443`
- L2 RPC URL: op-geth URL
- See Aggkit Configuration section above for detailed config file template

**Command to run Aggkit (Primary Instance)**:

```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,aggoracle
```

**Monitoring Aggkit with Prometheus**:

Aggkit exposes Prometheus metrics for monitoring the health and performance of the aggsender and aggoracle components. To enable metrics collection:

1. Configure the `[Prometheus]` section in your Aggkit config file (see Aggkit Configuration section above)
2. Set `Enabled = true` to expose the metrics endpoint
3. Configure your Prometheus server to scrape metrics from `http://<aggkit-host>:<prometheus-port>/metrics` (default port: 9091)

#### 6. Aggkit Bridge (Separate Instance)

Deploy the Aggkit bridge as a separate instance to provide the public bridge REST API. See the [Run Aggkit Bridge](./run-aggkit-bridge.md) runbook for complete deployment instructions.

**Key Points**:
- Run bridge as a separate instance from the primary Aggkit
- Bridge endpoint must be publicly accessible for Polygon's Bridge Hub
- Can scale horizontally based on API traffic

#### 7. Legacy Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `ghcr.io/0xpolygon/zkevm-bridge-service:v0.6.2`

**Startup Order**: Deploy after PostgreSQL and OP-Geth

**Dependencies**: PostgreSQL, OP-Geth

#### 8. Legacy Bridge UI

**Purpose**: Web interface for legacy bridge operations

**Docker Image**: `ghcr.io/0xpolygon/zkevm-bridge-ui:multi-network`

**Startup Order**: Deploy after Legacy Bridge Service

**Dependencies**: Legacy Bridge Service

## AggchainFEP
### OP Stack

#### 1. PostgreSQL Database

**Purpose**: Database for legacy bridge service

**Docker Image**: e.g., `bitnamisecure/postgresql@sha256:05f12b9dc62012ac6987bf3160241d2cbdeb60cf6d245f772d8582f89371929f`

**Startup Order**: Deploy first (dependency for other services)

#### 2. OP-Geth (Execution Layer)

**Purpose**: Ethereum execution client (L2)

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101602.0`

**Startup Order**: Deploy after secrets

**Dependencies**: Secrets

**Configuration**:
- Archive node type
- L2 chain ID and block time configuration
- JSON-RPC API endpoints
- Isthmus hard fork enabled

#### 3. OP-Node (Consensus Layer)

**Purpose**: Optimism consensus client

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.13.2`

**Startup Order**: Deploy after OP-Geth

**Dependencies**: OP-Geth, secrets

**Configuration**:
- Rollup configuration JSON
- L1 and L2 chain parameters
- System configuration
- Isthmus hard fork enabled

#### 4. OP-Batcher

**Purpose**: Batch submission to L1

**Docker Image**: `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.12.0`

**Startup Order**: Deploy after OP-Geth

**Dependencies**: OP-Geth, secrets

#### 5. OP Succinct Proposer

**Purpose**: Generates and manages proofs for L2 state transitions using OP Succinct proving system

**Docker Image**: `ghcr.io/agglayer/op-succinct/op-succinct:v3.1.0-agglayer`

**Startup Order**: Deploy after PostgreSQL database and secrets

**Dependencies**: PostgreSQL database, secrets, OP-Node, OP-Geth

#### 6. Aggkit Prover

**Purpose**: Manages proof generation requests and coordinates with OP Succinct proposer

**Docker Image**: `ghcr.io/agglayer/aggkit-prover:1.4.2`

**Startup Order**: Deploy after OP Succinct Proposer

**Dependencies**: OP Succinct Proposer, OP-Geth, OP-Node

#### 7. Aggkit (Primary Instance)

**Purpose**: Agglayer integration and oracle services

**Docker Image**: `ghcr.io/agglayer/aggkit:0.7.0`

**Startup Order**: Deploy after OP-Batcher, OP-Geth

**Dependencies**: OP-Batcher, OP-Geth

**Components**:
- `aggsender`: Certificate management
- `aggoracle`: Oracle services for OP Stack

> [!IMPORTANT]
> **Two-Instance Architecture**
>
> Starting with Aggkit v0.7.0, IPs should run **two separate Aggkit instances**:
> 1. **Primary Instance**: Runs `aggsender,aggoracle` - handles critical certificate submission and oracle services
> 2. **Bridge Instance**: Runs `bridge` only - provides public REST API
>
> This separation ensures that public bridge API traffic does not impact critical certificate submission operations. See the [Run Aggkit Bridge](./run-aggkit-bridge.md) runbook for bridge deployment details.

**Command to run Aggkit (Primary Instance)**:

```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,aggoracle
```

**Monitoring Aggkit with Prometheus**:

Aggkit exposes Prometheus metrics for monitoring the health and performance of the aggsender and aggoracle components. To enable metrics collection:

1. Configure the `[Prometheus]` section in your Aggkit config file (see Aggkit Configuration section in the CDK-Erigon section above)
2. Set `Enabled = true` to expose the metrics endpoint
3. Configure your Prometheus server to scrape metrics from `http://<aggkit-host>:<prometheus-port>/metrics` (default port: 9091)

> [!NOTE]
> The bridge component should be run in a separate instance. See [Run Aggkit Bridge](./run-aggkit-bridge.md) for details.

#### 8. Aggkit Bridge (Separate Instance)

Deploy the Aggkit bridge as a separate instance to provide the public bridge REST API. See the [Run Aggkit Bridge](./run-aggkit-bridge.md) runbook for complete deployment instructions.

**Key Points**:
- Run bridge as a separate instance from the primary Aggkit
- Bridge endpoint must be publicly accessible for Polygon's Bridge Hub
- Can scale horizontally based on API traffic

#### 9. Legacy Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `ghcr.io/0xpolygon/zkevm-bridge-service:v0.6.2`

**Startup Order**: Deploy after PostgreSQL and OP-Geth

**Dependencies**: PostgreSQL, OP-Geth

#### 10. Legacy Bridge UI

**Purpose**: Web interface for legacy bridge operations

**Docker Image**: `ghcr.io/0xpolygon/zkevm-bridge-ui:multi-network`

**Startup Order**: Deploy after Legacy Bridge Service

**Dependencies**: Legacy Bridge Service

## Component Version Matrix

### AggchainECDSAMultisig Versions
#### CDK-Erigon

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `bitnamisecure/postgresql@sha256:05f12b9dc62012ac6987bf3160241d2cbdeb60cf6d245f772d8582f89371929f` |
| Pool Manager | `ghcr.io/0xpolygon/zkevm-pool-manager:v0.1.2` |
| CDK-Erigon | `ghcr.io/0xpolygon/cdk-erigon:v2.61.23` |
| Aggkit | `ghcr.io/agglayer/aggkit:0.7.0` |
| Legacy Bridge | `ghcr.io/0xpolygon/zkevm-bridge-service:v0.6.2` |
| Legacy Bridge UI | `ghcr.io/0xpolygon/zkevm-bridge-ui:multi-network` |

#### OP Stack

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `bitnamisecure/postgresql@sha256:05f12b9dc62012ac6987bf3160241d2cbdeb60cf6d245f772d8582f89371929f` |
| OP-Geth | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101602.0` |
| OP-Node | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.13.2` |
| OP-Batcher | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.11.5` |
| Aggkit | `ghcr.io/agglayer/aggkit:0.7.0` |
| Legacy Bridge | `ghcr.io/0xpolygon/zkevm-bridge-service:v0.6.2` |
| Legacy Bridge UI | `ghcr.io/0xpolygon/zkevm-bridge-ui:multi-network` |

### AggchainFEP Versions
#### OP Stack

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `bitnamisecure/postgresql@sha256:05f12b9dc62012ac6987bf3160241d2cbdeb60cf6d245f772d8582f89371929f` |
| OP-Geth | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101602.0` |
| OP-Node | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.13.2` |
| OP-Batcher | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.12.0` |
| OP Succinct Proposer | `ghcr.io/agglayer/op-succinct/op-succinct:v3.1.0-agglayer` |
| Aggkit Prover | `ghcr.io/agglayer/aggkit-prover:1.4.2` |
| Aggkit | `ghcr.io/agglayer/aggkit:0.7.0` |
| Legacy Bridge | `ghcr.io/0xpolygon/zkevm-bridge-service:v0.6.2` |
| Legacy Bridge UI | `ghcr.io/0xpolygon/zkevm-bridge-ui:multi-network` |

## Deployment Considerations

### Resource Requirements
#### AggchainECDSAMultisig

**CDK-Erigon Components:**
- CDK-Erigon Sequencer/RPC: High CPU and memory requirements
- Pool Manager: Moderate resources
- Storage: 500Gi for blockchain data

**OP Stack Components:**
- OP-Geth: High CPU and memory (archive mode)
- OP-Node: Moderate resources
- OP-Batcher: Low to moderate resources

#### AggchainFEP
**OP Stack Components:**
- OP-Geth: High CPU and memory (archive mode)
- OP-Node: Moderate resources
- OP-Batcher: Low to moderate resources
- OP Succinct Proposer: High CPU and memory for proof generation
- Aggkit Prover: Moderate to high CPU and memory
- Storage: 500Gi for blockchain data, proof data, and certificate data

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

### AggchainECDSAMultisig Issues
#### CDK-Erigon

1. **Sequencer Sync Issues**: Check L1 connectivity and contract addresses
2. **Pool Manager Connection**: Verify database connectivity
3. **Certificate Submission**: Check Agglayer connectivity and credentials

#### OP Stack

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
