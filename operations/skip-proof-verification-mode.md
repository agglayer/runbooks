# Run a Chain in `skip-proof-verification` Mode

## Introduction

`skip-proof-verification` is an `aggkit-prover` mode for **non-production FEP chains** that keeps the
full settlement path alive — aggsender → aggkit-prover → op-succinct-proposer → agglayer → L1 — while
removing the cost of real SP1 proving.

In this mode the prover:

1. asks `op-succinct-proposer` for an aggregation proof as usual (the proposer runs in SP1 mock mode);
2. **executes** the standard aggchain program on the host with FEP proof verification disabled, so the
   bridge constraints, the L1-head inclusion against the L1 Info Tree and the aggchain params are all
   evaluated;
3. **proves** the `noop` program — a real SP1 proof of an empty program that simply commits the public
   values it is given.

The certificate therefore carries the noop selector `0xFFFF0001` instead of the standard one.

What is and is not enforced once the mode is live:

|                Check                  |            Status             |
| :-----------------------------------: | :---------------------------: |
| FEP execution proof                   | **not verified**              |
| Range + aggregation proofs            | mock                          |
| Bridge constraints                    | checked (host execution only) |
| FEP L1-head inclusion in L1 Info Tree | checked (host execution only) |
| Aggchain params vs contract           | checked (host execution only) |
| SP1 proof reaching L1                 | noop program — proves nothing |

> [!WARNING]
> **Nothing in this mode is cryptographically proven.** The checks above are enforced by the prover
> process on trust; the SP1 proof that reaches L1 attests only that the public values were committed.
> Use this on devnets and testnets only. Never register the noop vkey on the `AgglayerGateway`.

---

## Prerequisites

- **`aggkit-prover` ≥ `2.2.0-rc.3`** (`ghcr.io/agglayer/aggkit-prover:2.2.0-rc.3`).
- **`op-succinct-proposer` ≥ `v3.14.1`**.
- Access to the **Aggchain manager** of the `AggchainFEP` contract (EOA key or multisig).
- `cast` CLI ([Foundry](https://book.getfoundry.sh/getting-started/installation)).
- `jq`.

---

## Setup

```shell
export rollup_address=0x              # AggchainFEP proxy
export l1_rpc_url=https://
export aggchain_manager_key=0x        # or use your multisig flow
```

---

## 1. Collect the vkeys and selectors

Run against the **exact prover image** you will deploy:

```shell
aggkit-prover vkey                 # standard aggchain vkey
aggkit-prover vkey-selector        # standard selector, e.g. 0x000B0001
aggkit-prover vkey --noop          # noop vkey
aggkit-prover vkey-selector --noop # 0xFFFF0001
```

```shell
export noop_vkey=0x
export noop_selector=0xFFFF0001
export standard_vkey=0x
export standard_selector=0x000B0001
```

> [!NOTE]
> The noop selector reserves aggchain vkey version `0xFFFF`, so it can never collide with a real
> program version. The last 2 bytes remain the `AGGCHAIN_TYPE` (`0x0001` for FEP).

---

## 2. Register **both** owned vkeys

`addOwnedAggchainVKey` reverts with `OwnedAggchainVKeyAlreadyAdded` if a key is already set — e.g. on a
chain coming out of [recovery](./recover-halted-fep-chain.md). Check each selector first:

```shell
cast call $rollup_address "ownedAggchainVKeys(bytes4)(bytes32)" $noop_selector     --rpc-url $l1_rpc_url
cast call $rollup_address "ownedAggchainVKeys(bytes4)(bytes32)" $standard_selector --rpc-url $l1_rpc_url
```

|        Owned vkey         |          Action           |
| :-----------------------: | :-----------------------: |
| `0x000…000`               | `addOwnedAggchainVKey`    |
| equals the expected vkey  | skip                      |
| different                 | `updateOwnedAggchainVKey` |

```shell
cast send --private-key $aggchain_manager_key $rollup_address \
  "addOwnedAggchainVKey(bytes4,bytes32)" $noop_selector $noop_vkey --rpc-url $l1_rpc_url

cast send --private-key $aggchain_manager_key $rollup_address \
  "addOwnedAggchainVKey(bytes4,bytes32)" $standard_selector $standard_vkey --rpc-url $l1_rpc_url
```

> [!IMPORTANT]
> Registering the **standard** vkey as well is not optional in practice. Step 3 disables the gateway
> defaults for *every* selector, so without it the chain can no longer settle in `mode = "standard"` —
> you would have to toggle `enableUseDefaultVkeysFlag()` / `disableUseDefaultVkeysFlag()` on every mode
> change. Owning both keeps in-flight certificates and the way back working.

See [Add Owned Aggchain VKey](./add-owned-aggchain-vkey.md) for the full semantics of these calls.

---

## 3. Disable the default vkeys

```shell
cast call $rollup_address "useDefaultVkeys()(bool)" --rpc-url $l1_rpc_url
```

Only if it returns `true` (the call reverts with `UseDefaultVkeysAlreadyDisabled` otherwise):

```shell
cast send --private-key $aggchain_manager_key $rollup_address \
  "disableUseDefaultVkeysFlag()" --rpc-url $l1_rpc_url
```

---

## 4. Disable optimistic mode (if enabled)

```shell
cast call $rollup_address "optimisticMode()(bool)" --rpc-url $l1_rpc_url
```

If `true`, disable it — see [Toggle Optimistic Mode](./optimistic-mode.md). Leaving it on defeats the
purpose: the prover would never call `op-succinct-proposer`, and the path this mode is meant to
exercise would stay untested.

> [!WARNING]
> After disabling optimistic mode the op-succinct DB must be flushed, because existing proven ranges
> may not match the last-proven block the aggsender requests. Do it in step 5, **with the proposer
> stopped** — flushing a running real-mode proposer lets it rebuild the ranges with paid proofs.

---

## 5. Switch `op-succinct-proposer` to mock mode — **before** the prover

1. **Stop aggkit.** It stays stopped until step 6 completes, so no certificate request lands while the
   proposer and the prover run in mismatched modes.
2. **Stop `op-succinct-proposer`.**
3. **Flush the op-succinct DB** (required if optimistic mode was disabled in step 4).
4. Set mock mode in the proposer's **deployment environment** (env file, Helm values, …) and start it:

```env
OP_SUCCINCT_MOCK=true
```

   If you launch the proposer from a shell instead, use `export OP_SUCCINCT_MOCK=true` — a bare
   assignment is not passed to the process.

Confirm its startup line reports `mock: true`:

```
INFO Validity proposer configuration loaded  … range_proof_interval=1800 submission_interval=1800 mock=true …
```

> [!IMPORTANT]
> Order matters. The prover sends every proof request to `op-succinct-proposer` and fetches an SP1
> **mock** aggregation proof, whatever the prover's own config says. If the proposer is still in real
> mode when the prover switches, those requests start real, **paid** aggregation proofs and no
> certificate comes out.

> [!NOTE]
> Since [provers#374](https://github.com/agglayer/provers/pull/374) the prover connects to the proposer
> **lazily** — there is no reachability probe at startup, and it logs a deferred connection instead. A
> misconfigured or missing proposer therefore does not fail fast: the prover starts cleanly and every
> request fails afterwards. Verify from the per-request logs, not from a clean startup.

---

## 6. Switch `aggkit-prover` to `skip-proof-verification`

```toml
[aggchain-proof-service.aggchain-proof-builder]
mode = "skip-proof-verification"
```

Keep a **real** prover type (`network-prover` or `cpu-prover`): the noop program still needs a real SP1
proof, because the agglayer verifies the aggchain proof for real. `mock-prover` is accepted but warns,
and the resulting certificate can only be verified by a mock verifier.

**Restart the prover.** The config is read at startup only — a config change without a restart silently
keeps the old mode.

**Then start aggkit.**

---

## 7. Verify

Prover startup must show both the selector and the proposer mode:

```
WARN Skip-proof-verification mode: the standard program is executed without verifying the FEP proof,
     only the noop program is proven   noop_vkey=0x…
INFO Aggchain vkey selector in effect  vkey_selector=0xffff0001  proposer_mode=mock
```

Per-certificate, the prover should log a host execution followed by the proof:

```
INFO Received GenerateAggchainProof request  last_proven_block=… requested_end_block=…
INFO Program executed without proof  instructions=…              <- standard program executed
INFO Aggchain proof generated  last_proven_block=… end_block=…
```

On-chain, decode the `aggchainData` of a settlement and confirm word 0 is the noop selector:

```shell
cast tx <settlement_tx> --rpc-url $l1_rpc_url --json | jq -r .input | tail -c 193
```

|  Word  |         Field          |            Expected             |
| :----: | :--------------------: | :-----------------------------: |
| `[0]`  | `aggchainVKeySelector`  | `0xFFFF0001`                    |
| `[1]`  | `outputRoot`            | claim root of the end block     |
| `[2]`  | `l2BlockNumber`         | end block of the certificate    |

Then confirm settlement keeps advancing:

```shell
cast call $rollup_address "latestOutputIndex()(uint256)"  --rpc-url $l1_rpc_url
cast call $rollup_address "latestBlockNumber()(uint256)"  --rpc-url $l1_rpc_url
```

---

## Troubleshooting

<details>
<summary><b><code>No completed range proofs found for the requested range</code></b></summary>

<br>

```
ERROR Unable to execute GenerateAggchainProof request
  last_proven_block=… requested_end_block=…
  ProposerServiceError(… Grpc(Status { code: NotFound,
    message: "No completed range proofs found for the requested range" }))
```

This is logged at `ERROR` but is **usually normal**. The proposer selects range proofs with
`status = Complete`, matching commitments, and `end_block <=` the L2 safe head **as derived at an L1
block 20 blocks old**. Two distinct causes:

|           Symptom            |                            Meaning                            |             Action             |
| :--------------------------: | :-----------------------------------------------------------: | :----------------------------: |
| `highest_contiguous_proven_block` **below** the last settled block | The proposer genuinely has not finished the range | Wait                           |
| `highest_contiguous_proven_block` **at or above** the last settled block | The proof exists; you are waiting on op-batcher to post the covering batches to L1 and for that L1 block to age | Wait — can exceed 30 min with lumpy batching |

Read the proposer's own metrics line to tell them apart:

```
INFO unrequested=0 prove=0 execution=0 witness_generation=0
     highest_contiguous_proven_block=… latest_proposed_block=…
```

> [!NOTE]
> Do not treat `highest_contiguous_proven_block` advancing as proof that the next aggregation request
> will succeed — it is computed without the commitment filter that the aggregation query applies.

Expect a burst of these errors for one range immediately after any mode switch: a change of
`OpSuccinctConfig` makes the proposer log *"Deleted all unrequested, execution, and witness generation
requests and canceled all prove requests with different commitment configs"*, and the current range is
rebuilt from scratch.

</details>

<details>
<summary><b>Certificates still carry the standard selector after the switch</b></summary>

<br>

The prover did not pick up the config. Confirm `Aggchain vkey selector in effect` appears with
`vkey_selector=0xffff0001` in the logs *after* your change; if not, the pod was not restarted.

</details>

<details>
<summary><b>Settlement reverts on the aggchain vkey</b></summary>

<br>

`useDefaultVkeys` is still `true`, or the selector in use has no owned vkey registered. Re-check steps
2 and 3.

</details>

<details>
<summary><b>The <code>rollupConfigHash</code> or <code>rangeVkeyCommitment</code> changed</b></summary>

<br>

A proposer upgrade can change them. They must also be updated in the `AggchainFEP` `OpSuccinctConfig`,
otherwise `mode = "standard"` will no longer settle. Derive the expected values from the op-succinct
ELFs you are running and compare them against the contract:

```shell
aggkit-prover op-succinct-vkey --elf-dir <dir with aggregation-elf and range-elf-embedded>
```

This prints a ready-to-paste `[aggchain-proof-service.op-succinct]` section. The prover takes these
vkeys from `op-succinct-elfs` or from the config — it does **not** fetch them at runtime
([provers#375](https://github.com/agglayer/provers/pull/375) reverted that). Cross-check with the
proposer's `Program configuration loaded` line and the contract:

```shell
name=$(cast call $rollup_address "selectedOpSuccinctConfigName()(bytes32)" --rpc-url $l1_rpc_url)
cast call $rollup_address "opSuccinctConfigs(bytes32)(bytes32,bytes32,bytes32)" $name --rpc-url $l1_rpc_url
```

</details>

---

## Switching back to `mode = "standard"`

Reverse order, with aggkit paused throughout — otherwise a request arriving while the proposer is
real and the prover is still in `skip-proof-verification` starts a paid aggregation proof that is then
discarded:

1. **Stop aggkit.**
2. Stop `op-succinct-proposer`, **flush the op-succinct DB**, set `OP_SUCCINCT_MOCK=false` in its environment, and start it.
   Wait for it to produce real range proofs.
3. Set `mode = "standard"` and restart `aggkit-prover`.
4. Confirm `Aggchain vkey selector in effect  vkey_selector=0x000b0001  proposer_mode=network`.
5. **Start aggkit.**
6. Leave `useDefaultVkeys = false` as long as you own both vkeys. Only call
   `enableUseDefaultVkeysFlag()` if you want to return to the gateway defaults, and wait for any
   in-flight noop certificate to settle first.

---

## References

- [Recover a Halted FEP Chain After an L2 Reorg](./recover-halted-fep-chain.md)
- [Add Owned Aggchain VKey](./add-owned-aggchain-vkey.md)
- [`agglayer/provers`](https://github.com/agglayer/provers) — prover releases and the `mode` reference in its README
- [`agglayer/succinct-cost-reports`](https://github.com/agglayer/succinct-cost-reports) — SP1 cost extraction
