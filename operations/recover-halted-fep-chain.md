# Recover a Halted FEP Chain After an L2 Reorg

## Introduction

When an L2 reorgs **past a block whose output is already settled on L1**, the chain halts permanently:
the `AggchainFEP` contract keeps hashing the orphaned output root as the pre-root, while the prover
builds its proof from the live L2 pre-root. Every certificate then fails on the agglayer with
`Aggchain hash mismatch`, and no amount of retrying helps — settled outputs cannot be rewritten.

`mode = "recovery"` is the way out. It proves a single certificate whose pre-root is taken **from L1**
rather than from L2, re-anchoring the chain onto the settled state so normal settlement can resume.

> [!WARNING]
> In `recovery` the aggchain proof verifies **nothing** — not the FEP, not the bridge constraints. The
> certificate is accepted because it is proven with the `noop` program under a selector the rollup owns.
> Non-production chains only. Never register the noop vkey on the `AgglayerGateway`.

---

## Confirm the diagnosis first

Do not run this runbook on a chain that is merely slow. Confirm all three:

**1. The agglayer reports `Aggchain hash mismatch`:**

```shell
cast rpc --rpc-url $agglayer_rpc interop_getLatestKnownCertificateHeader $network_id | jq .
```

```json
"status": { "InError": { "error": { "InternalError":
  "Aggchain hash mismatch. from l1: 0x16ec…47fe, from certificate: 0xf395…e398." } } }
```

**2. The settled output root does not match what the L2 reports for that block.** Compare the L1
output against the L2 node:

```shell
idx=$(cast call $rollup_address "latestOutputIndex()(uint256)" --rpc-url $l1_rpc_url)
cast call $rollup_address "getL2Output(uint256)" $idx --rpc-url $l1_rpc_url
blk=$(cast call $rollup_address "latestBlockNumber()(uint256)" --rpc-url $l1_rpc_url)
cast rpc optimism_outputAtBlock $(cast to-hex $blk) --rpc-url $l2_op_node_rpc | jq -r .outputRoot
```

A mismatch here is the reorg. Run the same check on a **second, independently synced L2 node** before
concluding — a single unhealthy node can produce a false positive (see
[Troubleshooting](#the-l2-node-cannot-serve-state-at-the-anchor)).

**3. The local exit roots still agree.** This decides whether you need LET recovery first:

```shell
cast call $rollup_manager "rollupIDToRollupData(uint32)" $network_id --rpc-url $l1_rpc_url  # lastLocalExitRoot
cast call $l2_bridge_address "getRoot()(bytes32)" --rpc-url $l2_rpc_url
```

If they differ, reconcile with [Backward/Forward LET recovery](./backward-forward-let-recovery.md)
**before** continuing. If they match, proceed.

---

## Prerequisites

- **`aggkit-prover` ≥ `2.2.0-rc.3`** (`ghcr.io/agglayer/aggkit-prover:2.2.0-rc.3`).
- Access to the **Aggchain manager** of the `AggchainFEP` contract.
- An L2 execution node that can serve the **end block** of the recovery range.
- `cast` CLI ([Foundry](https://book.getfoundry.sh/getting-started/installation)).

```shell
export rollup_address=0x              # AggchainFEP proxy
export rollup_manager=0x
export network_id=
export l1_rpc_url=https://
export l2_rpc_url=https://
export agglayer_rpc=https://
export aggchain_manager_key=0x
```

---

## 1. Collect the noop vkey and selector

Run against the **exact prover image** you will deploy:

```shell
aggkit-prover vkey --noop
aggkit-prover vkey-selector --noop    # 0xFFFF0001
```

```shell
export noop_vkey=0x
export noop_selector=0xFFFF0001
```

---

## 2. Register the noop vkey and disable the default vkeys

```shell
cast send --private-key $aggchain_manager_key $rollup_address \
  "addOwnedAggchainVKey(bytes4,bytes32)" $noop_selector $noop_vkey --rpc-url $l1_rpc_url

cast send --private-key $aggchain_manager_key $rollup_address \
  "disableUseDefaultVkeysFlag()" --rpc-url $l1_rpc_url
```

> [!IMPORTANT]
> `disableUseDefaultVkeysFlag()` switches the rollup off the `AgglayerGateway` defaults for **every**
> selector, not just the noop one. Before returning to `mode = "standard"` you must also register the
> standard vkey — see [step 6](#6-return-to-mode--standard). Register both now if you want to avoid a
> second contract call later.

See [Add Owned Aggchain VKey](./add-owned-aggchain-vkey.md) for the full semantics.

---

## 3. Drop the aggsender's cached proof

The stuck certificate is `InError` and the aggsender holds a cached aggchain proof for it. Until that
is dropped, it keeps resending the same certificate and never calls the prover again.

**Delete the aggsender DB only — not the syncer DBs** — then restart aggkit and let the aggsender
resync from the agglayer.

> [!WARNING]
> Deleting the bridge/GER syncer DBs forces a full L2 resync and is not needed here.

---

## 4. Run the prover in `recovery`

```toml
[aggchain-proof-service.aggchain-proof-builder]
mode = "recovery"
```

Keep a **real** prover type (`network-prover` or `cpu-prover`) — the noop program still needs a real
SP1 proof, because the agglayer verifies the aggchain proof for real.

`op-succinct-proposer` is not used in this mode and can stay stopped — the prover never calls it, and
since [provers#374](https://github.com/agglayer/provers/pull/374) it performs no reachability probe at
startup either.

**Restart the prover.** The config is read at startup only. Confirm:

```
INFO Recovery mode: op-succinct-proposer is not used
WARN Recovery mode: nothing is verified, the pre-root is taken from the latest L1 output  noop_vkey=0x…
INFO Aggchain vkey selector in effect  vkey_selector=0xffff0001  proposer_mode=none
```

---

## 5. Watch the recovery certificate settle

The prover logs the inputs it assembled, all taken from L1:

```
INFO Retrieving chain data for the recovery aggchain proof
     last_proven_block=1778860 end_block=5005167 l1_pre_root=0x736fc5a0…fc42
INFO Recovery aggchain-params unrolled values: AggchainParamsValues { l2_pre_root: 0x736fc5a0…fc42,
     claim_root: …, claim_block_num: 5005167, rollup_config_hash: …, optimistic_mode: true,
     trusted_sequencer: …, range_vkey_commitment: …, aggregation_vkey_hash: … }
```

One certificate carries the whole gap — the jump from the stale anchor to the current chain can be
millions of blocks. Verify on L1:

```shell
cast call $rollup_address "latestOutputIndex()(uint256)"  --rpc-url $l1_rpc_url
cast call $rollup_address "latestBlockNumber()(uint256)"  --rpc-url $l1_rpc_url
```

`latestBlockNumber` should jump to the recovery `end_block`, and the settlement's `aggchainData` word 0
should be `0xFFFF0001`.

> [!NOTE]
> `recovery` is self-chaining: each subsequent certificate anchors on the previous L1 output, so the
> mode keeps producing certificates indefinitely with no constraints verified. Move off it as soon as
> the chain is re-anchored.

---

## 6. Return to `mode = "standard"`

```shell
aggkit-prover vkey                 # standard aggchain vkey
aggkit-prover vkey-selector        # e.g. 0x000B0001
```

```shell
cast send --private-key $aggchain_manager_key $rollup_address \
  "addOwnedAggchainVKey(bytes4,bytes32)" $standard_selector $standard_vkey --rpc-url $l1_rpc_url
```

Then:

1. Start `op-succinct-proposer` (real mode, `OP_SUCCINCT_MOCK=false`) and let it build range proofs
   from the new `latestBlockNumber`.
2. Set `mode = "standard"` and restart the prover.
3. Confirm `Aggchain vkey selector in effect  vkey_selector=0x000b0001  proposer_mode=network`.

Once a few certificates have settled in `standard`, you may move to the cheaper
[`skip-proof-verification` mode](./skip-proof-verification-mode.md).

> [!NOTE]
> Leave `useDefaultVkeys = false` as long as you own both vkeys. Only call `enableUseDefaultVkeysFlag()`
> to return to the gateway defaults, and wait for any in-flight noop certificate to settle first.

---

## Troubleshooting

<details>
<summary><b><code>optimisticMode</code> and the recovery certificate</b></summary>

<br>

The prover reads `optimisticMode()` from the contract and hashes it into the aggchain params, exactly
as the contract does. You may leave optimistic mode on or off during recovery — but if you toggle it
mid-recovery, the change takes effect on the **next** certificate, not retroactively.

</details>

<details>
<summary><b>The L2 node cannot serve state at the anchor</b></summary>

<br>

```
InvalidPreBlockSketchFinalization(TrieFromProofError(MismatchedStateRoot(found, expected)))
```

Seen in `standard`/`skip-proof-verification`, not in `recovery` — the recovery path builds no pre-block
state sketch. If you hit it *before* switching to recovery, the L2 execution node is serving
`eth_getProof` responses inconsistent with its own block headers.

A quick, decisive health check:

```shell
cast rpc eth_getProof $l2_bridge_address '[]' latest --rpc-url $l2_rpc_url
```

A node that cannot serve state for its own head (`no state found for block number …`) is resyncing or
rebuilding and will hand the prover inconsistent history. A **different reconstructed root on every
retry** means the node is faulty; a stable mismatch means a genuine divergence. Restore a healthy,
consistent archive node — or sync a replica from scratch and promote it — before diagnosing a reorg.

</details>

<details>
<summary><b><code>Recovery mode: request anchored at block X but the latest L1 output is at block Y</code></b></summary>

<br>

The prover refuses any request not anchored at the latest L1 output. Either the aggsender still holds
the stale cached proof (redo [step 3](#3-drop-the-aggsenders-cached-proof)), or another certificate
settled in between — let the aggsender resync and retry.

</details>

<details>
<summary><b><code>Recovery end block X is not after the last proven block Y</code></b></summary>

<br>

The aggsender requested a range that does not advance the chain. Usually transient while the aggsender
catches up after the DB reset.

</details>

<details>
<summary><b><code>Aggchain hash mismatch</code> persists after recovery</b></summary>

<br>

Compare the prover's logged `AggchainParamsValues` against the contract field by field. Every value
except `claim_root` and `claim_block_num` must come from L1 — if one does not match, the prover and the
contract disagree about the op-succinct config or the trusted sequencer.

</details>

---

## References

- [Run a Chain in `skip-proof-verification` Mode](./skip-proof-verification-mode.md)
- [Add Owned Aggchain VKey](./add-owned-aggchain-vkey.md)
- [Backward/Forward LET recovery](./backward-forward-let-recovery.md)
- [Toggle Optimistic Mode](./optimistic-mode.md)
- [`agglayer/provers`](https://github.com/agglayer/provers) — prover releases and the `mode` reference

First exercised on `bali-84-op` (rollup 84, Bali) on 2026-10-07: one recovery certificate carried the
chain from L2 block 1,778,860 to 5,005,167 after a 36-day halt.
