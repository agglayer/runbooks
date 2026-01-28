# Aggkit Upgrade: v0.7 to v0.8

Upgrade instructions for Aggkit from v0.7.x to v0.8.0.

## Version

- **From**: `ghcr.io/agglayer/aggkit:0.7.x`
- **To**: `ghcr.io/agglayer/aggkit:0.8.0`

## Configuration Changes

v0.8.0 reorganizes several top-level parameters into appropriate subsections and adds new retry configuration:

| v0.7 Parameter | v0.8 Parameter | Change |
|----------------|----------------|--------|
| `AgglayerURL` (top-level) | `[AggSender.AgglayerClient.GRPC].URL` | Moved |
| `NetworkID` (top-level) | Removed | Removed |
| `polygonBridgeAddr` (top-level) | `[L1Config].BridgeAddr` + `[L2Config].BridgeAddr` | Moved & renamed |

**Required changes:**
1. Remove: `AgglayerURL`, `NetworkID`, `polygonBridgeAddr` from top level
2. Add: `[AggSender.AgglayerClient.GRPC].URL` with your Agglayer URL
3. Add: `[L1Config].BridgeAddr` and `[L2Config].BridgeAddr`
4. Add to `[AggSender]`: `RetryCertAfterInError = true` (retry certificates in error state)
5. Add to `[AggSender]`: `CheckStatusCertificateInterval = "1m"` (check certificate status interval)

See the complete v0.8 config example in [Run L2 Trusted Environment](../operations/run-l2-trusted-environment.md#5-aggkit-primary-instance).

## New Features

### Fast L1 Deposits

v0.8.0 introduces support for faster L1 to L2 deposits by allowing IPs to configure reduced block finality requirements:

**Configuration**: Set `L1InfoTreeSync` and `BridgeL1Sync` block finality to `LatestBlock/-6` to enable:
- L1→L2 bridge completion in ~6 L1 blocks (~70 seconds)
- Significantly faster deposit experience compared to using `FinalizedBlock`

**Trade-offs**:
- **Faster deposits**: Users see deposits bridged in ~70s instead of waiting for L1 finalization (~15 minutes)
- **Reorg risk**: Reduced finality increases risk of chain reorganizations affecting bridge state

**Example configuration**:
```toml
[L1InfoTreeSync]
BlockFinality = "LatestBlock/-6"

[BridgeL1Sync]
BlockFinality = "LatestBlock/-6"
```

> [!NOTE]
> This configuration is optional and should be evaluated based on your network's risk tolerance. The default `FinalizedBlock` setting prioritizes safety over speed.

## Upgrade Steps

1. **Backup** your current config.toml
2. **Update config** with the changes above
3. **Stop** Aggkit v0.7.x instance
4. **Update image** to `ghcr.io/agglayer/aggkit:0.8.0`
5. **Start** Aggkit with updated config
6. **Verify** logs show successful startup and certificate submissions

No database resync required.
