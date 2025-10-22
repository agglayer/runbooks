# Aggkit Upgrade: v0.5 to v0.7.0

This document provides instructions for upgrading Aggkit from version v0.5.* to version v0.7.0.

## Overview

Aggkit v0.7.0 introduces several improvements and configuration changes. This upgrade is applicable to both CDK-Erigon and OP Stack deployments.

## Prerequisites

### Migration Path Considerations

The upgrade process depends on your current Aggkit version:

- **From v0.5.4 or later**: Direct upgrade to v0.7.0 is supported
- **From v0.5.3 or earlier**: A full resync of the Aggkit bridge database is required

### For Migrations from v0.5.3 or Earlier

If you are upgrading from v0.5.3 or any earlier version, you must perform a bridge database resync to ensure compatibility with v0.7.0. Follow this process:

1. **Keep your current Aggkit instance running** (v0.5.3 or earlier) to maintain aggsender operations
2. **Deploy a second Aggkit v0.7.0 instance** in parallel with **only the bridge component**:
   ```bash
   aggkit run --cfg=/etc/aggkit/config-v0.7.toml --components=bridge
   ```
   > [!IMPORTANT]
   > Do NOT run a second aggsender instance. Only run the bridge component to avoid conflicts.

3. **Wait for the bridge sync to complete** - Monitor the logs to ensure the v0.7.0 bridge has fully synced data from L1
4. **Switch to v0.7.0 with aggsender** - Once sync is complete, update the v0.7.0 instance to include aggsender:
   ```bash
   aggkit run --cfg=/etc/aggkit/config-v0.7.toml --components=aggsender,bridge
   ```
5. **Remove the old Aggkit instance** (v0.5.3 or earlier) after verifying v0.7.0 is working correctly

This parallel deployment approach ensures zero downtime for certificate submission while the bridge database syncs.

## Version Information

- **From**: `ghcr.io/agglayer/aggkit:0.5.x`
- **To**: `ghcr.io/agglayer/aggkit:0.7.0`

## Upgrade Steps

### Step 1: Update Configuration

Update your Aggkit configuration file to include new v0.7.0 fields. The configuration schema remains backward compatible with most v0.5 settings.

Ensure your configuration includes the following bridge-related sections:

```toml
# L1 reorg detector configuration
[ReorgDetectorL1]
FinalizedBlock = LatestBlock

# L1 bridge sync configuration
[BridgeL1Sync]
BlockFinality = LatestBlock

# REST API configuration (bridge)
[REST]
Port = "5577"
```

> [!NOTE]
> For the complete Aggkit v0.7.0 configuration reference, see the **[Aggkit Configuration section](../operations/run-l2-trusted-environment.md#5-aggkit)** in the Run L2 Trusted Environment runbook.

### Step 2: Update Aggkit Version

Update your Aggkit deployment to version `v0.7.0`:

```
ghcr.io/agglayer/aggkit:0.7.0
```

### Step 3: Run Aggkit

Run Aggkit with your updated configuration:

**For CDK-Erigon Stack:**
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,bridge
```

**For OP Stack:**
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,aggoracle,bridge
```

## Critical Requirements

> [!IMPORTANT]
> **Public Bridge Endpoint Requirement**
>
> The Aggkit bridge REST API endpoint **MUST be publicly exposed and accessible** on the Internet. This is a critical requirement because Polygon's Bridge Hub application needs to poll bridge data from your endpoint.
>
> Ensure that:
> - The bridge endpoint is accessible via a public URL (e.g., `https://bridge.yourchain.com`)
> - Firewall rules allow external access to the bridge REST API port (default: 5577)
> - SSL/TLS certificates are properly configured for HTTPS access
> - The endpoint has sufficient bandwidth and uptime guarantees
>
> Failure to properly expose this endpoint will prevent users from being able to use Polygon's Bridge Hub to interact with your chain's bridge.

## Rollback Procedure

If you encounter issues with v0.7.0, rollback to v0.5.x:

1. Restore your configuration backup (remove v0.7.0-specific sections)
2. Change the image version back to `ghcr.io/agglayer/aggkit:0.5.x`
3. Restart Aggkit

## Troubleshooting

If you encounter issues:

- **Configuration errors**: Check Aggkit logs for missing or incorrect configuration parameters
- **Bridge API not accessible**: Ensure firewall rules allow access to port 5577 and verify the public endpoint is properly configured

## Additional Resources

- [Run L2 Trusted Environment](../operations/run-l2-trusted-environment.md) - Full deployment guide
- [Run Aggsender Committee](../operations/run-aggsender-committee.md) - Multisig aggsender-committee setup
- [Run Aggoracle Committee](../operations/run-aggoracle-committee.md) - Multisig aggoracle-committee setup

