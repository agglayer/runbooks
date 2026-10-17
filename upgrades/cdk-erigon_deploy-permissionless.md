# Deploy zkEVM Permissionless Node

This document provides instructions for deploying a permissionless zkEVM node using CDK-Erigon. Permissionless nodes allow anyone to run a full node that syncs with the zkEVM network without requiring special permissions or access to trusted infrastructure components.

## Overview

A permissionless zkEVM node:
- Syncs blockchain state from the network
- Provides RPC access to the zkEVM network
- Does not participate in block production (sequencing)
- Can be used for read operations, transaction broadcasting, and local development

This guide covers deployment for both:
- **Cardona**: Polygon zkEVM testnet
- **zkEVM**: Polygon zkEVM mainnet

## Prerequisites

Before deploying a permissionless node, ensure you have:

1. **Docker**: Docker installed and running.
2. **Storage**: Sufficient disk space for Erigon's datadir.
3. **L1 RPC Endpoint**: Access to an Ethereum L1 RPC endpoint (required for L1 data verification)
   - **Cardona (testnet)**: Sepolia testnet RPC endpoint
   - **zkEVM (mainnet)**: Ethereum mainnet RPC endpoint
4. **Port Availability**: Ensure the RPC port (default 8545) is available or configure a different port
5. **Erigon Version**: Check for the latest release on the zkEVM repository. 

## Using Snapshots (Recommended)

Polygon provides daily snapshots that allow you to start syncing from a recent state rather than from genesis. This dramatically reduces the initial sync time from days to minutes.

**Snapshot URLs:**
- **Cardona Testnet**: `https://storage.googleapis.com/zkevm-testnet-snapshots/zkevm-testnet-erigon-snapshot.tgz`
- **zkEVM Mainnet**: `https://storage.googleapis.com/zkevm-mainnet-snapshots/zkevm-mainnet-erigon-snapshot.tgz`

## Deployment

The deployment instructions below use these snapshots by default. However, you can also run the Docker container directly without a snapshot (though this will take significantly longer to sync).

### Cardona

To deploy a zkEVM Cardona permissionless node, execute the following commands:

> [!NOTE]
> For Cardona testnet, you need a **Sepolia** testnet RPC endpoint for the `L1_RPC` variable.

```bash
# Create data directory
mkdir -p zkevm-cardona-pless

# Download latest snapshot
wget -c https://storage.googleapis.com/zkevm-testnet-snapshots/zkevm-testnet-erigon-snapshot.tgz

# Extract snapshot to data directory
tar xzvf zkevm-testnet-erigon-snapshot.tgz -C zkevm-cardona-pless

# Run the container
docker run --rm -d \
  --name zkevm-cardona-pless \
  -p $PORT:8545 \
  -v ./zkevm-cardona-pless:/datadir \
  hermeznetwork/cdk-erigon:$VERSION \
  --config="./cardona.yaml" \
  --datadir="/datadir" \
  --zkevm.l1-rpc-url=$L1_RPC
```

### Mainnet

To deploy a zkEVM permissionless node, execute the following commands:

> [!NOTE]
> For zkEVM mainnet, you need an **Ethereum mainnet** RPC endpoint for the `L1_RPC` variable.

```bash
# Create data directory
mkdir -p zkevm-mainnet-pless

# Download latest snapshot
wget -c https://storage.googleapis.com/zkevm-mainnet-snapshots/zkevm-mainnet-erigon-snapshot.tgz

# Extract snapshot to data directory
tar xzvf zkevm-mainnet-erigon-snapshot.tgz -C zkevm-mainnet-pless

# Run the container (it will continue syncing from the snapshot point)
docker run --rm -d \
  --name zkevm-mainnet-pless \
  -p $PORT:8545 \
  -v ./zkevm-mainnet-pless:/datadir \
  hermeznetwork/cdk-erigon:$VERSION \
  --config="./mainnet.yaml" \
  --datadir="/datadir" \
  --zkevm.l1-rpc-url=$L1_RPC
```

## Additional Resources

- [CDK-Erigon Documentation](https://github.com/0xPolygon/cdk-erigon)

## Notes

> [!NOTE]
> Snapshots provide a faster way to sync by starting from a recent state rather than syncing from genesis. The node will continue syncing from the snapshot point to the latest block. Polygon provides daily snapshots that are highly recommended to avoid days of initial sync time.

> [!NOTE]
> The configuration files (`cardona.yaml` and `mainnet.yaml`) are included in the Docker image and do not need to be downloaded separately.
