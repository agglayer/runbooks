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

## Upgrade Steps

1. **Backup** your current config.toml
2. **Update config** with the changes above
3. **Stop** Aggkit v0.7.x instance
4. **Update image** to `ghcr.io/agglayer/aggkit:0.8.0`
5. **Start** Aggkit with updated config
6. **Verify** logs show successful startup and certificate submissions

No database resync required.
