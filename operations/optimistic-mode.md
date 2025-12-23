# Toggle Optimistic Mode

Optimistic mode is a feature that allows the system to accept outputs without verification. This is useful for testing and development purposes, and as a fallback for `AggchainFEP` in the event of an outage.

When optimistic mode is enabled, the `aggkit-prover` service will skip Aggproofs and skip the verification of the FEP, only generating the BridgeProof as part of the Aggchain Proof, this proof will be sent to the Agglayer for verification included in a certificate.

## Enable Optimistic Mode

To enable optimistic mode, call the `enableOptimisticMode` function on the `AggchainFEP` contract.

```solidity
function enableOptimisticMode() external onlyOptimisticModeManager {
   if (optimisticMode) {
      revert OptimisticModeEnabled();
   }

   optimisticMode = true;
   emit EnableOptimisticMode();
}
```

Example using cast:

```
cast send --private-key [ROLLUP_ADMIN_KEY] [AGGCHAIN_FEP_CONTRACT_ADDRESS] "enableOptimisticMode()"
```


## Disable Optimistic Mode

By default, optimistic mode is disabled. To switch back to validity mode, call the `disableOptimisticMode` function on the `OPSuccinctL2OutputOracle` contract.

```solidity
   function disableOptimisticMode() external onlyOptimisticModeManager {
   if (!optimisticMode) {
      revert OptimisticModeNotEnabled();
   }

   optimisticMode = false;
   emit DisableOptimisticMode();
}
```

```
cast send --private-key [ROLLUP_ADMIN_KEY] [AGGCHAIN_FEP_CONTRACT_ADDRESS] "disableOptimisticMode()"
```

> [!warning]
> After disabling optimistic mode, the op-succinct DB needs to be flushed because the already existing proven ranges may not match the Last Proven block request coming from the AggSender.
> Pending range proofs need to be regenerated.
