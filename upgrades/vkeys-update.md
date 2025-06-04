# Update ELFs Procedure for Kurtosis

This document outlines the process for updating ELFs (Executable and Linkable Format files) when they are updated in the upstream op-succinct repository.

## Overview

When ELFs are updated in the upstream op-succinct repository, several components need to be synchronized to ensure compatibility across the entire system. This process involves updating dependencies, verification keys, and contract parameters.

## Prerequisites

- Access to the following repositories:
  - `agglayer/provers`
  - `agglayer/agglayer-contracts`
  - `agglayer/runbooks`
- Understanding of the deployment process
- Coordination with the deployment team

## Update Steps

### 1. Update ELFs Dependency in Prover

Update the `aggkit-prover` ELFs dependency in the [Cargo.toml file](https://github.com/agglayer/provers/blob/main/Cargo.toml):

```toml
op-succinct-elfs = { git = "https://github.com/agglayer/op-succinct.git", tag = "v2.2.1-agglayer" }
```

**Note**: Replace `v2.2.1-agglayer` with the actual tag version from the upstream update.

### 2. Update Contract Parameters

Update the verification keys in the AggchainFEP contract:

- **File**: [`contracts/v2/aggchains/AggchainFEP.sol`](https://github.com/agglayer/agglayer-contracts/blob/feature/ongoing-v0.3.0/contracts/v2/aggchains/AggchainFEP.sol#L72)
- **Parameters to update**:
  - `rangeVkeyCommitment`
  - `aggregationVkey`

These values must match the new ELFs to ensure proper verification.

### 3. Service Dependencies

**Important**: The agglayer and the Proof Producer (PP) do not directly depend on the op-succinct program versions thanks to the `aggchain_params` abstraction layer. This means these services will automatically use the updated parameters once the contracts are updated.

### 4. Coordinate Deployment

The deployment process requires careful coordination:

1. **Build new artifacts** with the updated ELFs
2. **Update contract values** before restarting services
3. **Follow the contract update procedure** detailed in the [parameter update runbook](https://github.com/agglayer/runbooks/blob/12acabb412c31e94d5ca7d4194c097f66c6d0ec7/upgrades/update-parameters.md)

## Important Notes

- **Timing**: Contract updates must be deployed before service restarts to avoid mismatched verification keys
- **Testing**: Ensure all components are tested with the new ELFs before production deployment
- **Communication**: Coordinate with the deployment team to schedule the update window

## Troubleshooting

If verification fails after the update:
1. Verify that all ELFs versions match across components
2. Check that contract parameters were updated correctly
3. Ensure services are using the latest contract state

## References

- [Provers Cargo.toml](https://github.com/agglayer/provers/blob/main/Cargo.toml)
- [AggchainFEP Contract](https://github.com/agglayer/agglayer-contracts/blob/feature/ongoing-v0.3.0/contracts/v2/aggchains/AggchainFEP.sol)
- [Parameter Update Runbook](https://github.com/agglayer/runbooks/blob/12acabb412c31e94d5ca7d4194c097f66c6d0ec7/upgrades/update-parameters.md)
