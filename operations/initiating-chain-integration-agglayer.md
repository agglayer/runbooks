# Initiating Chain Integration with the Agglayer

This document provides instructions for public users to propose and execute the `attachAggchainToAL` function from the `RollupManagerContract` using a Gnosis Safe multisig wallet.

## Overview

The `attachAggchainToAL` function allows authorized parties to attach an chain to the Agglayer (AL) infrastructure. This operation must be executed through a Gnosis Safe multisig for enhanced security and governance.

**Cardona Contract Address (Sepolia):** `0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff`

### User Access Levels

There are different types of users with varying levels of access to the Gnosis Safe multisig:

- **Full Signers/Owners**: Can propose, review, sign, and execute transactions
- **Proposers Only**: Can propose transactions but cannot sign or execute them
  - This typically includes **Polygon IPs (Implementation Providers)** and other authorized public users
  - These users must rely on full signers to review, approve, and execute their proposed transactions

## Prerequisites

Before proceeding, ensure you have:

1. **Gnosis Safe Access**: 
   - **For Proposers**: Access to propose transactions to the Gnosis Safe multisig
   - **For Signers**: Must be a signer/owner of the Gnosis Safe multisig
2. **Required Threshold**: (For execution) Sufficient signers must be available to meet the Safe's signature threshold
3. **Network Access**: Connection to the Sepolia testnet
4. **Function Parameters**: All required parameters for the `attachAggchainToAL` function
5. **Coordination**: (For proposers) Coordination with Safe signers for transaction review and approval

## Function Parameters

The `attachAggchainToAL` function requires the following parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `rollupTypeID` | `uint32` | The unique identifier of the rollup type to be attached |
| `chainID` | `uint64` | The chain ID of the chain |
| `initializeBytesAggchain` | `bytes` | Initialization data for the chain |

> [!NOTE]
> Please verify the exact function signature and parameters on [Etherscan](https://sepolia.etherscan.io/address/0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff#writeProxyContract#F4) before execution.

## Step-by-Step Instructions

### For Proposers Only (e.g., Polygon IPs)

#### 1. Access the Gnosis Safe Interface

1. Navigate to [Gnosis Safe App](https://app.safe.global/)
2. Connect your wallet and access the appropriate Safe
3. Ensure you're connected to the **Sepolia testnet**

#### 2. Propose the Transaction

1. In the Safe interface, click on **"New Transaction"**
2. Select **"Contract Interaction"**
3. Enter the contract address: `0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff`
4. The interface should automatically load the contract ABI and available functions. If the contract is detected as a proxy, you must select "Implementation ABI" to access the correct functions.

> [!TIP]
> If the contract ABI fails to load automatically, you can manually add it by:
> 1. Clicking on the implementation contract link on Etherscan
> 2. Scrolling down to find the "Contract ABI" section
> 3. Copying the entire ABI
> 4. In the Safe interface, clicking "Add ABI" and pasting the copied ABI

#### 3. Configure the Function Call

1. From the function dropdown, select **`attachAggchainToAL`**
2. Fill in the required parameters:
   - **rollupTypeID**: Enter the rollup type identifier (uint32)
   - **chainID**: Enter the chain ID for the chain (uint64)
   - **initializeBytesAggchain**: Enter the initialization data as hex-encoded bytes

> [!IMPORTANT]
> When selecting the `rollupTypeID`, visit [Etherscan](https://sepolia.etherscan.io/address/0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff#readProxyContract#F28) to find the number of rollupTypeIds and get details about each rollupTypeId [here](https://sepolia.etherscan.io/address/0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff#readProxyContract#F29). Review the rollupTypeIds from the latest one going backwards until you find the appropriate one based on these criteria:
> - For chains using Pessimistic Proofs (PP) with OP-FEP Aggchain proofs (CONSENSUS_TYPE = 1 and AGGCHAIN_TYPE = 1): Select a rollupTypeId with `rollupVerifierType = 2`
> - For chains using PP to prove valid ECDSA signatures of certificates (CONSENSUS_TYPE = 0): Select a rollupTypeId with `rollupVerifierType = 1`
> - For legacy chains using zkEVM-prover proofs without PP: Select a rollupTypeId with `rollupVerifierType = 0`
> - Ensure the selected rollupTypeId definition does not have `obsolete = true`
> - Verify the `forkId` in the rollupTypeId definition matches your requirements

> [!IMPORTANT]
> The `initializeBytesAggchain` parameter requires encoding the following fields in order:
> - Chain admin address (address)
> - Chain sequencer address (address)
> - Chain gas token address (address, use zero address for ETH)
> - URL to the trusted RPC node run with the sequencer (string)
> - Chain name (string)
>
> You can encode these parameters using one of the following methods:

<details>
<summary>Using JavaScript</summary>

```javascript
const ethers = require('ethers');

/**
 * Function to encode the initialize bytes for pessimistic or state transition rollups
 * @param {String} admin Admin address
 * @param {String} trustedSequencer Trusted sequencer address
 * @param {String} gasTokenAddress Indicates the token address in mainnet that will be used as a gas token
 * @param {String} trustedSequencerURL Trusted sequencer RPC URL
 * @param {String} networkName Chain name
 * @returns {String} encoded value in hexadecimal string
 */
function encodeInitializeBytes(
    admin,
    sequencer,
    gasTokenAddress,
    sequencerURL,
    networkName,
) {
    return ethers.AbiCoder.defaultAbiCoder().encode(
        ['address', 'address', 'address', 'string', 'string'],
        [
            admin,
            sequencer,
            gasTokenAddress,
            sequencerURL,
            networkName,
        ],
    );
}
```

Decode with:

```javascript
function decodeTuple(tuple: any): [string, string, string, string, string] | null {
  try {
    const values = abiCoder.decode(["address", "address", "address", "string", "string"], tuple);
    return values;
  } catch (error) {
    console.error("Error decoding the tuple:", error);
    return null;
  }
}
```

</details>

<details>
<summary>Using cast (Foundry)</summary>

```shell
# The sed command removes the first 4 bytes (function signature) from the output
$ cast calldata "test(address,address,address,string,string)" \
  <adminAddress> \
  <sequencerAddress> \
  <gasTokenAddress> \
  "<trustedSequencerRPCURL>" \
  "<chainName>" | sed 's/^\(0x\)[0-9a-fA-F]\{8\}/\1/'
```

Decode with:

```shell
# The bytes below represent the complete calldata including the function signature,
# which is the raw output from the cast command before any sed processing
$ cast calldata-decode 'test(address,address,address,string,string)' <bytes>
```

</details>


#### 4. Submit Proposal

1. **Review all parameters carefully** - incorrect values could cause the transaction to fail or have unintended consequences
2. Check the estimated gas fees
3. Click **"Review"** to see the transaction summary
4. If everything looks correct, click **"Submit"** to create the transaction proposal
5. **Share the transaction link** with the Safe signers for review and approval

### For Full Signers/Owners

#### 5. Review and Sign the Proposal

1. Access the Safe interface and navigate to pending transactions
2. Locate the `attachAggchainToAL` transaction proposal
3. **Carefully review all parameters** and verify they are correct
4. If approved, sign the transaction
5. Coordinate with other required signers to collect sufficient signatures

#### 6. Execute the Transaction

1. Once the required threshold of signatures is collected, any signer can execute the transaction
2. Click **"Execute"** on the transaction
3. Confirm the execution in your wallet
4. Wait for the transaction to be mined on Sepolia

### Alternative: Direct Execution (For Full Signers Only)

If you are a full signer and want to propose and execute in one workflow:

1. Follow steps 1-3 from the "For Proposers Only" section
2. After submitting, immediately proceed to sign the transaction
3. Coordinate with other signers to collect required signatures
4. Execute the transaction once threshold is met

## Security Considerations

⚠️ **Important Security Notes:**

- **Double-check all parameters** before signing - blockchain transactions are irreversible
- **Verify the contract address** to ensure you're interacting with the correct contract
- **Ensure all signers understand** what the transaction will do
- **Keep transaction records** for audit and reference purposes
- **For proposers**: Provide clear documentation and context when sharing proposals with signers
- **For signers**: Carefully verify the proposer's identity and the transaction purpose before signing

## Communication Between Proposers and Signers

### For Proposers
When sharing a transaction proposal with signers, include:

1. **Parameter Details**: Explanation of each parameter value and its significance
2. **Timeline**: Any urgency or scheduling requirements
3. **Contact Information**: How signers can reach you for questions

### For Signers
Before signing a proposed transaction:

1. **Verify Proposer**: Confirm the proposer is authorized (e.g., verified Polygon IP)
2. **Review Parameters**: Ensure all parameter values are reasonable and expected
3. **Check Context**: Understand the business/technical justification for the attachment
4. **Validate Timing**: Confirm this is the appropriate time for this action
5. **Ask Questions**: Don't hesitate to request clarification before signing

## Verification

After successful execution, you can verify the transaction:

1. Check the transaction hash on [Sepolia Etherscan](https://sepolia.etherscan.io/)
2. Verify the transaction status shows "Success"
3. Check the contract state to confirm the chain has been properly attached
4. Monitor for any relevant events emitted by the contract

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Transaction fails with "Unauthorized" | Ensure the Safe has the necessary permissions to call this function |
| Gas estimation fails | Check that all parameters are valid and the function can be executed with current contract state |
| Parameters validation error | Verify all parameter types and values match the expected format |
| Insufficient signatures | Ensure enough Safe owners have signed to meet the threshold |
| Cannot propose transaction | Verify you have proposer access to the Safe; check with Safe administrators |
| Proposal not visible to signers | Ensure you've shared the correct transaction link with all required signers |
| Signers not responding | Follow up with Safe owners and provide clear context about the transaction purpose |

### Getting Help

If you encounter issues:

1. Check the [Gnosis Safe Documentation](https://docs.safe.global/)
2. Review the transaction details on Etherscan for error messages
3. Verify that your Safe has the necessary permissions
4. Consult with other Safe owners or technical team members

## Example Transaction

```solidity
// Function signature (Function ID: 0x97d289a3)
function attachAggchainToAL(
    uint32 rollupTypeID,
    uint64 chainID,
    bytes calldata initializeBytesAggchain
) external;
```

```javascript
// Example parameters (replace with actual values)
{
  "rollupTypeID": 1,
  "chainID": 1337,
  "initializeBytesAggchain": "0x1234567890abcdef..."
}
```

---

**Disclaimer**: This documentation is for educational purposes. Always verify the latest contract details and function signatures on Etherscan before executing any transactions. Ensure you understand the implications of the function call and have proper authorization before proceeding.
