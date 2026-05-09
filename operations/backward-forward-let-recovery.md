# Backward/Forward LET Recovery

Use this runbook when AggSender cannot progress certificates because the L1-settled Local Exit Root (LER) and the L2 bridge Local Exit Tree (LET) no longer line up.

The operator workflow is case-agnostic: do not choose a recovery type by hand. The `backward-forward-let` CLI diagnoses the state, computes the recovery plan, asks for confirmation, sends the required bridge-admin transactions, and verifies the final L2 state.

## Inputs

You need:

- The exact Aggkit config file for the affected deployment.
- A `backward-forward-let` binary from the same Aggkit version as the affected deployment.
- Network access to the L2 RPC, AggLayer gRPC, bridge-service REST API, and AggSender JSON-RPC configured for that deployment.
- Read-only AggLayer Admin API access only if the CLI reports missing certificate exits.
- Role credentials configured in the Aggkit config for:
  - `BackwardForwardLET.GERRemoverKey`
  - `BackwardForwardLET.EmergencyPauserKey`
  - `BackwardForwardLET.EmergencyUnpauserKey`
- The configured `BackwardForwardLET.L2NetworkID`, `BackwardForwardLET.BridgeServiceURL`, and `BackwardForwardLET.AggsenderRPCURL`.

Do not paste private keys, keystore passwords, or secret endpoint URLs into tickets or chat. Redact secret values from configs before escalation.

All examples put the global `--cfg` flag before subcommands:

```sh
export CFG=/etc/aggkit/config.toml

backward-forward-let --cfg "$CFG" cert-status
backward-forward-let --cfg "$CFG" --diagnose-only
```

If your deployment uses layered config files, keep every `--cfg` before the subcommand:

```sh
backward-forward-let --cfg /etc/aggkit/base.toml --cfg /etc/aggkit/recovery.toml cert-status
```

## Stop Conditions

Stop and collect the escalation data below if any of these happen:

- `backward-forward-let --help` does not show `--diagnose-only`, `--cert-exits-file`, and `cert-status`.
- Any command exits with an error you cannot directly resolve from this runbook.
- The diagnosis says certificate exits are missing and you cannot obtain an approved cert-ID map from the AggLayer admin owner, or cannot generate the AggLayer certificate file with the documented script.
- The diagnosis fails with `bridge service data not ready for recovery` after bridge-service indexing has had time to catch up.
- The recovery prompt or recovery plan is not what you expected.
- Any transaction fails, receipt status is not `1`, the CLI reports a final LER/deposit-count mismatch, or the bridge remains in emergency state.
- AggLayer keeps a certificate in `InError` after recovery and a follow-up certificate attempt.
- After recovery, AggSender reports it is running but stuck in startup or initial-status recovery, especially with `Manual recovery required: wipe the aggsender DB and restart aggsender`.
- After an AggSender DB wipe/restart, the target LER exists on L2 but `L2BridgeSyncer` is stalled or still cannot resolve that LER after catching up to the LET event block.

## Preflight

Create an evidence directory and capture the tool/version surface before any recovery command:

```sh
export CFG=/etc/aggkit/config.toml
export OUT="bfl-recovery-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$OUT"

backward-forward-let --version | tee "$OUT/version.txt"
backward-forward-let --help | tee "$OUT/help.txt"
backward-forward-let --cfg "$CFG" cert-status | tee "$OUT/cert-status.before.txt"
```

Representative `cert-status` output:

```text
Network ID: 1
Latest settled height: 42
Latest settled certificate ID: 0x8a2f000000000000000000000000000000000000000000000000000000000042
Latest settled LER: 0x4f6d000000000000000000000000000000000000000000000000000000000042
Latest settled deposit count: 128
Latest pending height: 43
Latest pending status: InError
Latest pending error: certificate PrevLocalExitRoot does not match settled local exit root
```

Before executing recovery, coordinate so no other operator or automation is sending recovery transactions. If you control the deployment, pause AggSender certificate submission until the CLI recovery has completed.

## Diagnose

Run diagnosis first. This prints the plan and stops before sending transactions:

```sh
backward-forward-let --cfg "$CFG" --diagnose-only | tee "$OUT/diagnosis.txt"
```

### Output: No Divergence

If the output says `NoDivergence`, stop. No recovery is needed.

```text
=== Backward/Forward LET Diagnosis ===

State                          LER                                                                Deposit Count
L1 Settled (AggLayer)          0x4f6d000000000000000000000000000000000000000000000000000000000042 128
L2 On-Chain (Bridge)           0x4f6d000000000000000000000000000000000000000000000000000000000042 128
L1 Settled Height:          42
L1 Settled Certificate ID:  0x8a2f000000000000000000000000000000000000000000000000000000000042

Case: NoDivergence — L1 settled state and L2 on-chain state are in sync.
Nothing to do: L1 settled state and L2 on-chain state are in sync.
```

### Output: Recovery Required

If the output says `Status: Recovery required`, review the plan but do not manually choose a recovery path. The CLI has already selected the required sequence.

```text
=== Backward/Forward LET Diagnosis ===

State                          LER                                                                Deposit Count
L1 Settled (AggLayer)          0x1111000000000000000000000000000000000000000000000000000000000042 129
L2 On-Chain (Bridge)           0x2222000000000000000000000000000000000000000000000000000000000044 131
L1 Settled Height:          42
L1 Settled Certificate ID:  0x8a2f000000000000000000000000000000000000000000000000000000000042

Status: Recovery required - BackwardLET and ForwardLET recovery required, including replay of real L2 bridges
Divergence Point (matching leaf count): 128

Divergent L1-Settled Leaves (1):
  LeafType OriginNet  OriginAddr                                DestNet    DestAddr                                  Amount
  [0] 0        0          0x0000000000000000000000000000000000000000 1          0x1111111111111111111111111111111111111111 500

Extra Real L2 Bridges (2):
  LeafType OriginNet  OriginAddr                                DestNet    DestAddr                                  Amount
  [0] 0        0          0x0000000000000000000000000000000000000000 1          0x2222222222222222222222222222222222222222 200
  [1] 0        0          0x0000000000000000000000000000000000000000 1          0x3333333333333333333333333333333333333333 300

Undercollateralized Tokens (1):
  OriginNet  OriginAddr                                Amount
  0          0x0000000000000000000000000000000000000000 500

=== Recovery Plan ===
The following steps will be executed:
  1. BackwardLET:   roll back L2 bridge to DivergencePoint DC=128
  2. ForwardLET #1: inject 1 divergent leaf(ves) (agglayer but not L2)
  3. ForwardLET #2: replay 2 real L2 bridge(s) (L2 but not agglayer)
  4. Verify: confirm L2 LER matches L1 settled LER
Diagnose-only mode: no recovery transactions were sent.
```

### Output: Missing Certificate Exits

If AggSender cannot provide stored certificate bridge exits, the CLI stops before recovery. Do not continue with guessed or hand-typed bridge exits.

```text
Status: Missing certificate exits - recovery cannot continue yet.
WARNING: Aggsender RPC returned no bridge exit data for the following certificate heights.
Recovery cannot proceed until this data is provided.

Missing certificates (2 heights):
  Height 42      CertID: 0x8a2f000000000000000000000000000000000000000000000000000000000042  [ID auto-resolved]
  Height 41      CertID: UNKNOWN            [contact agglayer admin for cert ID]

NOTE: After an aggsender DB wipe, this missing range may span the full settled history
  (for example heights 0..latest). This is expected for the fallback path.
  Do not fetch large ranges one-by-one manually; use a script or ask the agglayer admin
  for a batch export of cert IDs / bridge exits when many heights are missing.

NOTE: For heights with UNKNOWN cert IDs, ask the agglayer admin to look up
  (network_id, height) in the agglayer's certificate_per_network_cf column family,
  or check aggsender submission logs for the certificate ID at that height.

Preferred batch export path:
  1. Ask the agglayer admin owner to resolve an authoritative cert ID map
     from agglayer state, then fetch raw admin_getCertificate responses:
     {
       "network_id": <L2NetworkID>,
       "certificates": {
         "42": "0x8a2f000000000000000000000000000000000000000000000000000000000042",
         "41": "<CertID>"
       }
     }
  2. Store the raw agglayer responses in an agglayer certificate file:
     {
       "network_id": <L2NetworkID>,
       "certificates": {
         "<height>": {"jsonrpc":"2.0","result":[<Certificate>, <CertificateHeader|null>]}
       }
     }

The --cert-exits-file loader accepts either this raw agglayer certificate file
or the Aggkit-native heights-to-bridge_exits override format.

Manual admin API shape for each KNOWN cert ID:
  POST http://<agglayer-admin-url>/
  Content-Type: application/json

  {"jsonrpc":"2.0","method":"admin_getCertificate","params":["<CertID>"],"id":1}

  The response is [Certificate, CertificateHeader|null].
  It can be stored directly under the matching height key in --cert-exits-file.

Re-run the tool with:
  backward-forward-let --cfg <config> --cert-exits-file <path-to-agglayer-certificates.json>

No recovery transactions were sent.
Provide the missing certificate exits with --cert-exits-file, then rerun diagnosis.
```

Follow the fallback flow below, then rerun diagnosis with the generated AggLayer certificate file.

### Output: Bridge-Service Data Not Ready

If bridge-service has not indexed the L2 bridge data needed for recovery, the CLI exits before printing a recovery plan.

```text
Error: diagnosis failed: collect extra L2 bridges: bridge service data not ready for recovery: missing L2 bridge at DC=130; wait for bridge-service indexing and rerun diagnosis
```

Wait for bridge-service indexing to catch up, then rerun:

```sh
backward-forward-let --cfg "$CFG" --diagnose-only | tee "$OUT/diagnosis.retry.txt"
```

If the same deposit count remains missing after indexing should be complete, stop and escalate with the failed command output and bridge-service endpoint details.

## Fallback: Certificate Exits File

Use `--cert-exits-file` only when the CLI reports missing certificate exits. The file may contain raw AggLayer `admin_getCertificate` responses keyed by certificate height, or pre-extracted `bridge_exits` in Aggkit's native fallback format. The CLI uses it only as fallback for heights AggSender cannot serve.

The approved path is:

1. Send the missing height range and network ID to the AggLayer admin owner.
2. The AggLayer admin owner resolves those heights to certificate IDs from AggLayer state and exports the raw AggLayer certificates with the script below.
3. Re-run diagnosis with the generated AggLayer certificate file as `--cert-exits-file`.

### AggLayer Admin: Export Certificates

The fallback data must come from AggLayer state, not from guessed IDs or hand-written bridge exits.

Inputs from the recovery operator:

- `network_id`: the `BackwardForwardLET.L2NetworkID` from the affected Aggkit config.
- Missing certificate heights from the diagnosis output.
- The read-only AggLayer Admin API URL.
- Any required Admin API auth header, for example `Authorization: Bearer <token>`.

For each missing height, resolve `(network_id, height) -> certificate_id` from the AggLayer state DB `certificate_per_network_cf` mapping or an owner-approved equivalent export. The cert-ID map must have this shape:

```json
{
  "network_id": 1,
  "certificates": {
    "42": "0x8a2f000000000000000000000000000000000000000000000000000000000042",
    "41": "0x7b1e000000000000000000000000000000000000000000000000000000000041"
  }
}
```

Then run this shell script to fetch the raw AggLayer `admin_getCertificate` responses and write the file that Aggkit can load directly:

```sh
#!/usr/bin/env bash
set -Eeuo pipefail

export AGGLAYER_ADMIN_URL="${AGGLAYER_ADMIN_URL:?set read-only AggLayer Admin API URL}"
export CERT_IDS_FILE="${CERT_IDS_FILE:?set path to cert-ids.json}"
export AGGLAYER_CERTS_FILE="${AGGLAYER_CERTS_FILE:?set output path for agglayer-certificates.json}"
# Optional, for IAP or other protected admin endpoints:
# export AGGLAYER_ADMIN_AUTH_HEADER="Authorization: Bearer $JWT"

NETWORK_ID="$(jq -r '.network_id' "$CERT_IDS_FILE")"
test "$NETWORK_ID" != "null" -a "$NETWORK_ID" != "0"

tmp="$(mktemp)"
trap 'rm -f "$tmp" "$tmp.next"' EXIT

jq -n \
  --argjson network_id "$NETWORK_ID" \
  '{network_id: $network_id, description: "raw agglayer admin_getCertificate responses", certificates: {}}' \
  > "$tmp"

while IFS= read -r height; do
  cert_id="$(jq -r --arg h "$height" '.certificates[$h]' "$CERT_IDS_FILE")"
  jq -e -n --arg cert_id "$cert_id" '$cert_id | test("^0x[0-9a-fA-F]{64}$")' >/dev/null

  request="$(jq -n --arg cert_id "$cert_id" \
    '{jsonrpc:"2.0", id:1, method:"admin_getCertificate", params:[$cert_id]}')"

  curl_headers=(-H "Content-Type: application/json")
  if [ -n "${AGGLAYER_ADMIN_AUTH_HEADER:-}" ]; then
    curl_headers+=(-H "$AGGLAYER_ADMIN_AUTH_HEADER")
  fi

  response="$(curl -fsS -X POST "$AGGLAYER_ADMIN_URL" "${curl_headers[@]}" -d "$request")"

  jq -e \
    --argjson network_id "$NETWORK_ID" \
    --argjson height "$height" \
    '.error == null
     and .result[0].network_id == $network_id
     and .result[0].height == $height
     and (.result[0].bridge_exits | type == "array")' \
    <<<"$response" >/dev/null

  jq --arg h "$height" --argjson response "$response" \
    '.certificates[$h] = $response' "$tmp" > "$tmp.next"
  mv "$tmp.next" "$tmp"
done < <(jq -r '.certificates | keys[]' "$CERT_IDS_FILE" | sort -n)

mv "$tmp" "$AGGLAYER_CERTS_FILE"
trap - EXIT

jq -e \
  --argjson network_id "$NETWORK_ID" \
  '.network_id == $network_id
   and (.certificates | type == "object")
   and ([.certificates[] | (.result[0].network_id == $network_id and (.result[0].bridge_exits | type == "array"))] | all)' \
  "$AGGLAYER_CERTS_FILE" >/dev/null

sha256sum "$CERT_IDS_FILE" "$AGGLAYER_CERTS_FILE"
```

The output file is intentionally AggLayer-shaped. `backward-forward-let` accepts this format directly:

```json
{
  "network_id": 1,
  "description": "raw agglayer admin_getCertificate responses",
  "certificates": {
    "42": {
      "jsonrpc": "2.0",
      "id": 1,
      "result": [
        {
          "network_id": 1,
          "height": 42,
          "bridge_exits": []
        },
        null
      ]
    }
  }
}
```

Hand off `AGGLAYER_CERTS_FILE` through the approved incident channel, plus the command output, checksums, state DB source used for the cert-ID lookup, network ID, exported height range, and operator/ticket reference. Do not include AggLayer Admin API tokens, database credentials, or private endpoint secrets.

Run diagnosis with the AggLayer certificate file:

```sh
export CERT_EXITS_FILE=/secure/path/agglayer-certificates.json

backward-forward-let --cfg "$CFG" \
  --cert-exits-file "$CERT_EXITS_FILE" \
  --diagnose-only | tee "$OUT/diagnosis.with-cert-exits.txt"
```

The CLI validates the file network ID, each certificate height, and each certificate network ID, then extracts `bridge_exits` internally. Empty bridge-exit lists are preserved as `[]`. Stop if any of those checks fail.

Alternative Aggkit-native override shape:

```json
{
  "network_id": 1,
  "description": "generated from agglayer admin_getCertificate",
  "heights": {
    "42": [
      {
        "leaf_type": 0,
        "token_info": {
          "origin_network": 0,
          "origin_token_address": "0x0000000000000000000000000000000000000000"
        },
        "dest_network": 1,
        "dest_address": "0x1111111111111111111111111111111111111111",
        "amount": "500",
        "metadata": null
      }
    ],
    "41": []
  }
}
```

This format is still accepted, but the AggLayer admin handoff should prefer the raw `certificates` format above. Do not copy `admin_getCertificate` payloads into the `heights` shape by hand.

If the file is malformed, the CLI exits before recovery, for example:

```text
Error: load certificate exits override: agglayer certificates file /secure/path/agglayer-certificates.json: height key 42 does not match certificate height 43
```

If the cert-ID map is not authoritative or is incomplete, stop and return to the AggLayer admin owner. `backward-forward-let` validates and extracts certificate exits from provided AggLayer certificate data; it does not discover `(network_id,height) -> certID` itself.

Proceed only after diagnosis prints `Status: Recovery required` and a recovery plan.

## Recover

Wait until there is no open pending certificate before executing recovery:

```sh
backward-forward-let --cfg "$CFG" cert-status --wait-no-pending --timeout 30m \
  | tee "$OUT/cert-status.no-pending.txt"
```

Representative output:

```text
Network ID: 1
Latest settled height: 42
Latest settled certificate ID: 0x8a2f000000000000000000000000000000000000000000000000000000000042
Latest settled LER: 0x1111000000000000000000000000000000000000000000000000000000000042
Latest settled deposit count: 129
Latest pending certificate: none
Wait complete: no open pending certificate.
```

Run recovery interactively:

```sh
backward-forward-let --cfg "$CFG" | tee "$OUT/recovery.txt"
```

If using fallback exits:

```sh
backward-forward-let --cfg "$CFG" \
  --cert-exits-file "$CERT_EXITS_FILE" | tee "$OUT/recovery.txt"
```

The CLI asks for confirmation unless `--yes` is passed. Prefer the interactive prompt for manual operations.

Representative confirmation:

```text
=== Recovery Plan ===
The following steps will be executed:
  1. BackwardLET:   roll back L2 bridge to DivergencePoint DC=128
  2. ForwardLET #1: inject 1 divergent leaf(ves) (agglayer but not L2)
  3. ForwardLET #2: replay 2 real L2 bridge(s) (L2 but not agglayer)
  4. Verify: confirm L2 LER matches L1 settled LER

Proceed with recovery? [y/N] y
```

Representative successful completion:

```text
[step] Activating emergency state...
[tx] ActivateEmergencyState sent: 0xa100000000000000000000000000000000000000000000000000000000000001
[tx] ActivateEmergencyState confirmed in block 120001.
[step] Emergency state activated.
[step] BackwardLET: rolling back to DC=128...
[tx] BackwardLET sent: 0xb200000000000000000000000000000000000000000000000000000000000002
[tx] BackwardLET confirmed in block 120002.
[step] BackwardLET complete. DC=128
[step] ForwardLET (divergent leaves): inserting 1 leaf(ves)...
[tx] ForwardLET (divergent leaves) sent: 0xf100000000000000000000000000000000000000000000000000000000000001
[tx] ForwardLET (divergent leaves) confirmed in block 120003.
[step] ForwardLET (divergent leaves) complete. DC=129, LER=0x1111000000000000000000000000000000000000000000000000000000000042
[step] ForwardLET (extra L2 bridges): inserting 2 leaf(ves)...
[tx] ForwardLET (extra L2 bridges) sent: 0xf200000000000000000000000000000000000000000000000000000000000002
[tx] ForwardLET (extra L2 bridges) confirmed in block 120004.
[step] ForwardLET (extra L2 bridges) complete. DC=131, LER=0x3333000000000000000000000000000000000000000000000000000000000044
[verify] Final L2 state: DC=131, LER=0x3333000000000000000000000000000000000000000000000000000000000044
[verify] Final L2 state includes replayed L2 bridge data; rerun diagnosis after aggsender settles the follow-up certificate.
[step] Deactivating emergency state...
[tx] DeactivateEmergencyState sent: 0xd100000000000000000000000000000000000000000000000000000000000001
[tx] DeactivateEmergencyState confirmed in block 120005.
[step] Emergency state deactivated.

Recovery completed successfully.
```

If the operator declines the prompt, the CLI sends no transactions:

```text
Proceed with recovery? [y/N] n
Aborted.
```

## Verify

After recovery, coordinate AggSender certificate production with the deployment owner.

For normal recovery where AggSender was only paused and its local DB still matches the
latest AggLayer-settled certificate, resume AggSender certificate submission.

If this recovery follows an internal validation exercise or another deployment-specific
workflow that intentionally bypassed normal AggSender storage, follow that private
procedure for the AggSender resume handoff. Do not infer the DB path or service-control
command from this public runbook.

If recovery replayed real L2 bridge data, the CLI may print:

```text
[verify] Final L2 state includes replayed L2 bridge data; rerun diagnosis after aggsender settles the follow-up certificate.
```

This is an intermediate state, not full network recovery. Final `NoDivergence` depends on AggSender submitting and AggLayer settling the next normal certificate. Do not declare the network fully recovered until that follow-up certificate is `Settled` and a fresh diagnosis returns `NoDivergence`.

Check AggSender status through the deployment's configured JSON-RPC endpoint:

```sh
export AGGSENDER_RPC="<configured BackwardForwardLET.AggsenderRPCURL>"

curl -sS -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"aggsender_status","params":[]}' \
  "$AGGSENDER_RPC" | tee "$OUT/aggsender-status.after-recovery.json"
```

If `aggsender_status` shows `running=true` but the status is stuck in startup or initial-status recovery, and `last_error` says the local certificate differs from the AggLayer certificate or says to wipe the DB, stop and hand off to the deployment owner or DevOps. The required operation is:

```text
Wipe the affected deployment's AggSender local DB, then restart AggSender.
```

Do not improvise the DB path or service-control command from this public runbook. Record the operator, ticket, exact DB-wipe/restart command or runbook used, and timestamp in the evidence directory. After DevOps confirms the DB wipe and restart, rerun `aggsender_status`; it must leave the stale initial-status error before you continue polling for the follow-up certificate.

If the DB wipe clears the stale certificate error but AggSender remains in `starting_claim_syncer_stage`, check logs before polling indefinitely. After a DB wipe and restart, local bridge-sync state may still be catching up. During that catch-up window, AggSender can temporarily fail to resolve the latest settled certificate's `NewLocalExitRoot`, for example:

```text
failed to resolve the bridge exit block number for NewLocalExitRoot 0x...
failed to get local exit root by hash 0x...: not found
```

Do not treat that message as an immediate bridge-sync data-recovery blocker. First assert that the LER exists on L2:

- If the LER is the current root, check the L2 bridge `getRoot()` value.
- If the LER is historical, check the L2 bridge `BackwardLET` or `ForwardLET` transaction logs that created it.

If the LER does not exist on L2, stop; the expected post-recovery `NoDivergence` path is invalid and the incident needs engineering review.

If the LER exists on L2, wait for the local `L2BridgeSyncer` to catch up through the L2 block that emitted the relevant LET event. In logs, look for `module:L2BridgeSyncer` entries such as:

```text
block <l2-block-containing-let-event> processed with 1 events
```

While `L2BridgeSyncer` is advancing and has no errors, keep polling `aggsender_status` and the logs. The expected recovery signal is that the claim-syncer setup eventually succeeds and AggSender moves to `certificate_stage`, for example:

```text
Set next required block for claim syncer to <block>
successful run Setting next required block for claim syncer based on agglayer's latest settled certificate
Aggsender status changed to: certificate_stage
```

Stop and hand off to the deployment owner or DevOps only if the LER exists on L2 but `L2BridgeSyncer` errors, stalls before the LET block, processes the LET block without resolving the LER, or AggSender keeps reporting `failed to get local exit root by hash ...: not found` after bridge-sync catch-up. Capture `aggsender_status`, the relevant AggSender and `L2BridgeSyncer` logs, the latest settled certificate height and LER, the LET transaction/block evidence, and the ticket/operator reference before continuing.

Check certificate progress:

```sh
backward-forward-let --cfg "$CFG" cert-status --wait-no-pending --timeout 30m \
  | tee "$OUT/cert-status.after.txt"
```

If the successful recovery output said `Final L2 state matches L1 settled state`, rerun diagnosis immediately:

```sh
backward-forward-let --cfg "$CFG" --diagnose-only | tee "$OUT/diagnosis.after.txt"
```

Expected result:

```text
Case: NoDivergence — L1 settled state and L2 on-chain state are in sync.
Nothing to do: L1 settled state and L2 on-chain state are in sync.
```

If the successful recovery output said `Final L2 state includes replayed L2 bridge data`, wait for AggSender to submit and settle the follow-up certificate, then rerun diagnosis.

Certificate settlement can take a long time. Poll every few minutes and keep each sample in the evidence directory. A `Candidate` or `Pending` status at the expected height means settlement is still in progress; it is not a failure by itself. If logs show the certificate as `Proven` but `cert-status` still reports `Candidate`, keep polling `cert-status`; the final gate is `Requested height status: Settled`, followed by diagnosis returning `NoDivergence`.

```sh
export FOLLOW_UP_HEIGHT="$(awk '/Latest settled height:/ {print $4 + 1}' "$OUT/cert-status.no-pending.txt")"
test "$FOLLOW_UP_HEIGHT" -gt 0

for attempt in $(seq 1 24); do
  ts="$(date -u +%Y%m%dT%H%M%SZ)"
  backward-forward-let --cfg "$CFG" cert-status \
    --height "$FOLLOW_UP_HEIGHT" \
    | tee "$OUT/cert-status.follow-up-settlement-$ts.txt"

  if grep -q "Requested height status: Settled" "$OUT/cert-status.follow-up-settlement-$ts.txt"; then
    cp "$OUT/cert-status.follow-up-settlement-$ts.txt" "$OUT/cert-status.follow-up-settled.txt"
    break
  fi

  sleep 300
done

test -f "$OUT/cert-status.follow-up-settled.txt"

backward-forward-let --cfg "$CFG" --diagnose-only | tee "$OUT/diagnosis.after-follow-up.txt"
```

Expected settled output:

```text
Network ID: 1
Latest settled height: 43
Latest settled certificate ID: 0x8a2f000000000000000000000000000000000000000000000000000000000043
Latest settled LER: 0x3333000000000000000000000000000000000000000000000000000000000044
Latest settled deposit count: 131
Latest pending certificate: none
Requested height: 43
Requested height status: Settled
```

If the polling window expires, get operator approval before extending it. Stop if another certificate occupies the expected height or if the requested height reaches a terminal unexpected state such as `InError`.

## Escalation Data

Collect this before escalation:

- `$OUT/version.txt`, `$OUT/help.txt`, every `diagnosis*.txt`, every `cert-status*.txt`, and `recovery.txt` if recovery was attempted.
- The exact command that failed and its exit status.
- Environment name, Aggkit version, network ID, bridge-service URL name, AggSender RPC URL name, AggLayer gRPC endpoint name, and L2 RPC endpoint name. Do not include secrets.
- The config file path and a redacted copy of relevant non-secret config values.
- Last settled height, certificate ID, LER, deposit count, latest pending status, and latest pending error from `cert-status`.
- The diagnosis `Divergence Point`, divergent leaf count, extra L2 bridge count, and undercollateralized token summary.
- For missing certificate exits: the reported missing heights, which cert IDs were auto-resolved, which remained `UNKNOWN`, the owner/ticket for the authoritative cert-ID map, and checksums of the cert-ID map and AggLayer certificate file:

```sh
sha256sum "$CERT_IDS_FILE" | tee "$OUT/cert-ids-file.sha256"
sha256sum "$CERT_EXITS_FILE" | tee "$OUT/cert-exits-file.sha256"
```

- For bridge-service indexing failures: the missing `DC=<number>` from the error, when it was first seen, and whether it changed after retry.
- Any transaction hashes printed by the CLI and whether `DeactivateEmergencyState` completed.
