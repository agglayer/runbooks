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

**Docker Image**: `docker.io/bitnami/postgresql:16.2.1`

**Startup Order**: Deploy first (dependency for other services)

**Configuration**: Standard PostgreSQL setup

### 2. Pool Manager

**Purpose**: Transaction pool management for CDK-Erigon

**Docker Image**: `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/pool-manager:0.3.0`

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

### 5. AggKit (Agglayer Integration)

**Purpose**: Agglayer integration and certificate management

**Docker Image**: `ghcr.io/agglayer/aggkit:0.5.1`

**Startup Order**: Deploy after CDK-Erigon RPC

**Dependencies**: CDK-Erigon RPC

**Components**:
- `aggsender`: Sends certificates to Agglayer
- `bridge`: Bridge service component (when enabled)

**Configuration**:
- Agglayer client URL: `grpc-agglayer-dev.polygon.technology:443`
- Certificate send interval: `1m`
- Mode: `PessimisticProof`

### 6. Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge:v0.6.2-RC2`

**Startup Order**: Deploy after CDK-Erigon RPC

**Dependencies**: CDK-Erigon RPC

**Configuration**: L1 and L2 contract addresses, claim transaction management

### 7. Bridge UI (Optional)

**Purpose**: Web interface for bridge operations

**Docker Image**: `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge-ui:0.4.0`

**Startup Order**: Deploy after AggKit

**Dependencies**: AggKit

## Vanilla OP Stack

For networks using the standard OP Stack, the following components must be deployed:

### 1. PostgreSQL Database

**Purpose**: Database for bridge service

**Docker Image**: `docker.io/bitnami/postgresql:16.2.1`

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

**Configuration**:
- L2 OP-Geth URL: `http://op-geth:8545`
- OP-Node URL: `http://op-node:9545`

### 5. AggKit (Agglayer Integration)

**Purpose**: Agglayer integration and oracle services

**Docker Image**: `ghcr.io/agglayer/aggkit:dont-merge-disk-certs_2025_08_11_11_30_8e8657e`

**Startup Order**: Deploy after OP-Batcher and OP-Geth

**Dependencies**: OP-Batcher, OP-Geth

**Components**:
- `aggsender`: Certificate management
- `aggoracle`: Oracle services for OP Stack

**Configuration**:
- Agglayer client URL: `34.76.187.110:9089` (development)
- L2 RPC URL: `http://op-geth:8545`

### 6. Bridge Service

**Purpose**: Cross-chain asset transfers

**Docker Image**: `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge:0.7.0`

**Startup Order**: Deploy after PostgreSQL and OP-Geth

**Dependencies**: PostgreSQL, OP-Geth

### 7. Bridge UI (Optional)

**Purpose**: Web interface for bridge operations

**Docker Image**: `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge-ui:0.2.2`

**Startup Order**: Deploy after Bridge Service

**Dependencies**: Bridge Service

## Component Version Matrix

### CDK-Erigon Stack Versions

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `docker.io/bitnami/postgresql:16.2.1` |
| Pool Manager | `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/pool-manager:0.3.0` |
| CDK-Erigon | `docker.io/hermeznetwork/cdk-erigon:v2.61.23` |
| AggKit | `ghcr.io/agglayer/aggkit:0.5.1` |
| Bridge | `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge:v0.6.2-RC2` |
| Bridge UI | `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge-ui:0.4.0` |

### Vanilla OP Stack Versions

| Component | Docker Image |
|-----------|--------------|
| PostgreSQL | `docker.io/bitnami/postgresql:16.2.1` |
| OP-Geth | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-geth:v1.101503.1` |
| OP-Node | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:v1.12.0` |
| OP-Batcher | `us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:v1.11.5` |
| AggKit | `ghcr.io/agglayer/aggkit:dont-merge-disk-certs_2025_08_11_11_30_8e8657e` |
| Bridge | `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge:0.7.0` |
| Bridge UI | `europe-west2-docker.pkg.dev/prj-polygonlabs-shared-prod/polygonlabs-docker-prod/bridge-ui:0.2.2` |

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
