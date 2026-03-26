# Add Owned Aggchain VKey

Registers a new aggchain verification key on the rollup contract. This is required when a new prover version is released with a new `AGGCHAIN_VKEY_SELECTOR` (i.e. a new aggchain type + SP1 verifier version combination).

> [!IMPORTANT]
> The **Aggchain manager** address is required. This will typically be a multisig or EOA that controls the rollup. Confirm you control it before proceeding.

> [!WARNING]
> The function will revert if a VKey is already registered for the given selector. If you need to change an existing VKey, use `updateOwnedAggchainVKey` instead.

---

## Prerequisites

- The rollup must have `useDefaultVkeys` set to `false` for this runbook to take effect. When `useDefaultVkeys` is `true`, the aggchain VKey is fetched from the `AgglayerGateway` contract (managed by Polygon) and the `ownedAggchainVKeys` mapping is not used. You can disable the flag by calling `disableUseDefaultVkeysFlag()` with the **Aggchain manager**.
- The `AGGCHAIN VKEY` and `AGGCHAIN_VKEY_SELECTOR` values from the target [provers release](https://github.com/agglayer/provers/releases)
- Access to the **Aggchain manager** address (EOA private key or multisig)
- `cast` CLI installed ([Foundry](https://book.getfoundry.sh/getting-started/installation))

---

## Setup

Export env vars

```shell
export rollup_address=0x
export l1_rpc_url=https://
```

---

## Collect data from provers release

Go to [agglayer/provers releases](https://github.com/agglayer/provers/releases) and find the target release. Each release contains a table with the required values:

|          field           |                          example (v1.9.2)                          |
| :----------------------: | :----------------------------------------------------------------: |
|     `AGGCHAIN VKEY`      | `0x7767a8330ce68dac35265ba15d9eec6722b943cf00dc3b733779e1ae55696f70` |
| `AGGCHAIN_VKEY_SELECTOR` |                           `0x000a0001`                             |

Export them:

```shell
export aggchain_vkey=0x7767a8330ce68dac35265ba15d9eec6722b943cf00dc3b733779e1ae55696f70
export aggchain_vkey_selector=0x000a0001
```

> [!NOTE]
> The `aggchainVKeySelector` is a `bytes4` value composed of two parts:
> - First 2 bytes: `aggchainVKeyVersion` (e.g. `0x000a`)
> - Last 2 bytes: `AGGCHAIN_TYPE` (e.g. `0x0001`)

---

## Verify current state

Confirm you control the aggchainManager address:

```shell
cast call $rollup_address "aggchainManager()(address)" --rpc-url $l1_rpc_url
```

Confirm `useDefaultVkeys` is `false`:

```shell
cast call $rollup_address "useDefaultVkeys()(bool)" --rpc-url $l1_rpc_url
```

If it returns `true`, the rollup is using VKeys from the `AgglayerGateway` (managed by Polygon). You must first disable the flag by calling `disableUseDefaultVkeysFlag()` with the **Aggchain manager** before proceeding.

Check that the selector does **not** already have a VKey registered (should return `0x0000...`):

```shell
cast call $rollup_address "ownedAggchainVKeys(bytes4)(bytes32)" $aggchain_vkey_selector --rpc-url $l1_rpc_url
```

If this returns a non-zero value, the selector already has a VKey. Use `updateOwnedAggchainVKey` instead.

---

## Execute

Using the **Aggchain manager** address, call:

**EOA:**

```shell
cast send $rollup_address \
  "addOwnedAggchainVKey(bytes4,bytes32)" \
  $aggchain_vkey_selector \
  $aggchain_vkey \
  --rpc-url $l1_rpc_url \
  --private-key $PRIVATE_KEY
```

**Multisig:** generate the calldata and submit it through your multisig:

```shell
cast calldata "addOwnedAggchainVKey(bytes4,bytes32)" $aggchain_vkey_selector $aggchain_vkey
```

---

## Verify

Confirm the VKey was registered:

```shell
cast call $rollup_address "ownedAggchainVKeys(bytes4)(bytes32)" $aggchain_vkey_selector --rpc-url $l1_rpc_url
```

The returned value should match `$aggchain_vkey`.

You can also verify via the view function that resolves default vs owned keys:

```shell
cast call $rollup_address "getAggchainVKey(bytes4)(bytes32)" $aggchain_vkey_selector --rpc-url $l1_rpc_url
```

> [!NOTE]
> `getAggchainVKey` only returns the owned key when `useDefaultVkeys` is `false`. If it's still `true`, this call will return the gateway default key instead.
