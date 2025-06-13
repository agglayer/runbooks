# Unlocking op-succinct in Case of an Unprovable Block

This runbook outlines the procedure to recover `op-succinct` when the proposer gets stuck on a block that cannot be proven within the SP1 cluster. This may occur due to data availability issues, state mismatches, or bugs in the proof generation.

## Problem Context

When the proposer is attempting to prove a block range that is unprovable (due to errors or constraints in the SP1 proving stack), it may repeatedly fail and get stuck. To resume proving past the offending block, follow the steps below.

---

## Step-by-Step Recovery Procedure

1. **Stop the proposer**
   - Use your orchestration tool or CLI to gracefully shut down the `op-succinct` proposer process.

2. **Remove the failed block entry**
   - Manually delete or clear the internal record or cache of the failed block.
   - This ensures the proposer does not reattempt the same block upon restart.

3. **Enable Optimistic Mode**
   - Activate *Optimistic mode* to allow the chain to progress even without proof for the current block.
   - Monitor the system until the settlement moves past the offending block height.

4. **Disable Optimistic Mode**
   - Once a sufficient number of safe blocks have been finalized past the faulty block, turn off *Optimistic mode* to resume normal operation.

5. **Restart the proposer**
   - Relaunch the proposer service. It should now skip the unprovable block and continue proving subsequent blocks as expected.

---

## Expected Outcome

After following these steps, the proposer should bypass the offending block and resume regular proving operations.

---

## Notes & Recommendations

- Always validate the new settlement height before disabling Optimistic mode.
- Document the offending block hash/height for incident tracking.
- If multiple blocks are unprovable, consider deeper investigation into upstream issues.
