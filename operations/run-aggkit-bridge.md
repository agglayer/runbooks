# Run Aggkit Bridge

This runbook provides instructions for Implementation Providers (IPs) on deploying and configuring the Aggkit bridge service as a separate instance.

## Overview

The Aggkit bridge service should be run as a **separate instance** from the aggsender/aggoracle components. This separation ensures that critical components like the aggsender are not impacted by bridge API usage from public users.

**Architecture**:
- **Primary Aggkit Instance**: Runs `aggsender` (and `aggoracle` for OP Stack) - handles certificate submission to the Agglayer
- **Bridge Aggkit Instance**: Runs only the `bridge` component - provides public REST API for bridge queries

## Why Separate Instances?

Running the bridge as a separate instance provides:

1. **Resource Isolation**: Bridge API traffic from public users won't impact critical certificate submission
2. **Improved Reliability**: Issues with bridge service won't affect aggsender operations
3. **Security**: Better control over public-facing endpoints vs. internal components

## Prerequisites

- Aggkit v0.7.0 or later
- L1 and L2 RPC access
- Bridge contract addresses
- Public-facing infrastructure for the bridge REST API endpoint

## Configuration

The bridge instance requires a focused configuration with only bridge-related settings.

### Bridge Configuration Template

```toml
# Network ID of the L2 chain
NetworkID = 1

# Path to read/write data directory
PathRWData = "/data/bridge"

# L1 RPC URL for Ethereum connectivity
L1URL = "https://your-l1-rpc-endpoint.com"

# L2 RPC URL pointing to your execution client
L2URL = "http://execution-client:8123"

# Polygon bridge contract address on L2
polygonBridgeAddr = "0x0000000000000000000000000000000000000000"

# Block number where the rollup was created on L1
rollupCreationBlockNumber = 0
# Block number where the rollup manager was deployed on L1
rollupManagerCreationBlockNumber = 0
# Genesis block number on L1 (typically same value as rollupManagerCreationBlockNumber)
genesisBlockNumber = 0

# L1 chain configuration
[L1Config]
chainId = 11155111  # Sepolia testnet
polygonZkEVMGlobalExitRootAddress = "0x0000000000000000000000000000000000000000"
polygonRollupManagerAddress = "0x0000000000000000000000000000000000000000"
polTokenAddress = "0x0000000000000000000000000000000000000000"
# Rollup contract address
polygonZkEVMAddress = "0x0000000000000000000000000000000000000000"

# L2 chain configuration
[L2Config]
GlobalExitRootAddr = "0x0000000000000000000000000000000000000000"

# Logging configuration
[Log]
Environment = "production"
Level = "info"
Outputs = ["stderr"]

# RPC server configuration
[RPC]
Port = 5576

# L1 reorg detector configuration
[ReorgDetectorL1]
FinalizedBlock = "LatestBlock"

# L1 bridge sync configuration
[BridgeL1Sync]
BlockFinality = "LatestBlock"

# L1 info tree sync configuration
[L1InfoTreeSync]
SyncBlockChunkSize = 1000

# REST API configuration (bridge)
[REST]
Port = "5577"

# L2 GER sync configuration
[L2GERSync]
BlockFinality = "LatestBlock"
```

## Running the Bridge Instance

Start the bridge instance with only the `bridge` component:

```bash
aggkit run --cfg=/etc/aggkit/bridge-config.toml --components=bridge
```

> [!IMPORTANT]
> Do **NOT** run `aggsender` or `aggoracle` components in the bridge instance. These should only run in your primary Aggkit instance.

## Public Endpoint Requirements

> [!IMPORTANT]
> **Critical: Public Accessibility Required**
>
> The Aggkit bridge REST API endpoint **MUST be publicly exposed and accessible** on the Internet. This is a critical requirement because Polygon's Bridge Hub application needs to poll bridge data from your endpoint.
>
> Ensure that:
> - The bridge endpoint is accessible via a public URL (e.g., `https://bridge.yourchain.com`)
> - Firewall rules allow external access to the bridge REST API port (default: 5577)
> - SSL/TLS certificates are properly configured for HTTPS access
> - The endpoint has sufficient bandwidth and uptime guarantees
> - Load balancing is configured if running multiple bridge instances
>
> Failure to properly expose this endpoint will prevent users from being able to use Polygon's Bridge Hub to interact with your chain's bridge.

## Monitoring and Maintenance

### Health Checks

Monitor the bridge service health:

```bash
# Check bridge API health
curl https://your-bridge-endpoint.com/
```

## Security Best Practices

1. **Rate Limiting**: Implement rate limiting on the public bridge endpoint to prevent abuse
2. **DDoS Protection**: Use DDoS protection services for the public endpoint
3. **Monitoring**: Set up alerting for unusual traffic patterns or API errors
4. **Separate Networks**: Keep primary Aggkit instance on internal network, only bridge instance publicly accessible
5. **Regular Updates**: Keep Aggkit version up to date for security patches

## Initial Sync

When first deploying the bridge instance, it needs to sync historical bridge data from L1:

1. Start the bridge instance with the configuration
2. Monitor logs for sync progress
3. Initial sync time depends on:
   - Chain age and number of bridge transactions
   - L1 RPC performance
   - `SyncBlockChunkSize` setting
4. Bridge API will be available during sync, but data may be incomplete until sync completes

## Upgrading the Bridge Instance

When upgrading the bridge instance:

1. Update the Aggkit version in your deployment
2. Review configuration for any new bridge-related settings
3. Restart the bridge instance with the new version
4. Verify bridge API is accessible and responding correctly

For version-specific upgrade instructions, refer to the version-specific upgrade runbooks in the `upgrades/` directory.

## Additional Resources

- [Run L2 Trusted Environment](./run-l2-trusted-environment.md) - Full L2 deployment guide
- [Aggkit v0.5 to v0.7 Upgrade](../upgrades/aggkit-v0.5-to-v0.7.md) - Upgrade guide
