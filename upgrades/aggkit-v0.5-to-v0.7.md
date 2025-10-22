# Aggkit Upgrade: v0.5 to v0.7.0

This document provides instructions for upgrading Aggkit from version v0.5.* to version v0.7.0.

## Overview

Aggkit v0.7.0 introduces several improvements and configuration changes. This upgrade is applicable to both CDK-Erigon and OP Stack deployments.

## Prerequisites

### Migration Path Considerations

The upgrade process depends on your current Aggkit version:

- **From v0.5.4 or later**: Direct upgrade to v0.7.0 is supported
- **From v0.5.3 or earlier**: A full resync of the L1 info tree database is required

### For Migrations from v0.5.3 or Earlier

If you are upgrading from v0.5.3 or any earlier version, you must perform an L1 info tree database resync to ensure compatibility with v0.7.0.

The recommended approach is to:
1. Keep your current Aggkit instance running
2. Run a temporary v0.7.0 instance with only the `l1infotreesync` component to rebuild the L1 info tree database
   ```bash
   aggkit run --cfg=/etc/aggkit/config.toml --components=l1infotreesync
   ```
   > [!IMPORTANT]
   > Do NOT run a second aggsender instance. Only run the bridge component to avoid conflicts.
3. Once the resync completes, stop the old instance and update to v0.7.0 instance to include aggsender (and aggoracle for the op-stack)



## Version Information

- **From**: `ghcr.io/agglayer/aggkit:0.5.x`
- **To**: `ghcr.io/agglayer/aggkit:0.7.0`

## Upgrade Steps

### Step 1: Update Configuration

Update your Aggkit configuration file to include new v0.7.0 fields. The configuration schema remains backward compatible with all v0.5 settings.

> [!NOTE]
> For the complete Aggkit v0.7.0 configuration reference, see the **[Aggkit Configuration section](../operations/run-l2-trusted-environment.md)** in the Run L2 Trusted Environment runbook.

### Step 2: Update Aggkit Version

Update your Aggkit deployment to version `v0.7.0`:

```
ghcr.io/agglayer/aggkit:0.7.0
```

### Step 3: Run Aggkit

Run Aggkit v0.7.0 with your configuration:

**For CDK-Erigon Stack:**
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender
```

**For OP Stack:**
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,aggoracle
```

## Rollback Procedure

If you encounter issues with v0.7.0, rollback to v0.5.x:

1. Restore your configuration backup (remove v0.7.0-specific sections)
2. Change the image version back to `ghcr.io/agglayer/aggkit:0.5.x`
3. Restart Aggkit

## Troubleshooting

If you encounter issues:

- **Configuration errors**: Check Aggkit logs for missing or incorrect configuration parameters
- **Certificate submission failures**: Verify Agglayer connectivity and aggsender configuration

## Additional Resources

- [Run L2 Trusted Environment](../operations/run-l2-trusted-environment.md) - Full L2 deployment guide
- [Run Aggsender Committee](../operations/run-aggsender-committee.md) - Multisig aggsender-committee setup
- [Run Aggoracle Committee](../operations/run-aggoracle-committee.md) - Multisig aggoracle-committee setup

