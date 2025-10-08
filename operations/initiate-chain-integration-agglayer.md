# Initiate Chain Integration with the Agglayer

This document provides comprehensive instructions for integrating a chain with the Agglayer, covering both the chain attachment process and subsequent genesis file generation. The process involves two main phases:

1. **Chain Integration**: Proposing and executing the `attachAggchainToAL` function from the `AgglayerManager` contract (formerly RollupManager) using a Gnosis Safe multisig wallet
2. **Genesis Generation**: Creating the appropriate genesis file for your network based on the client type

## Overview

### Chain Integration Process

The `attachAggchainToAL` function allows authorized parties to attach a chain to the Agglayer (AL) infrastructure. This operation must be executed through a Gnosis Safe multisig for enhanced security and governance. Proposers can find the relevant Safe wallet addresses directly in the Safe's UI interface.

### Genesis File Generation

After successful chain integration, you'll need to generate a network genesis file. The procedure differs depending on whether your network uses CDK-Erigon or vanilla clients, with specific tools available for each type.

### User Access Levels

There are different types of users with varying levels of access to the Gnosis Safe multisig:

- **Full Signers/Owners**: Can propose, review, sign, and execute transactions
- **Proposers Only**: Can propose transactions but cannot sign or execute them
  - This typically includes **Polygon IPs (Implementation Providers)** and other authorized public users
  - These users must rely on full signers to review, approve, and execute their proposed transactions

## Prerequisites

### For Chain Integration

Before proceeding with the chain attachment, ensure you have:

1. **Gnosis Safe Access**:
   - **For Proposers**: Access to propose transactions to the Gnosis Safe multisig
   - **For Signers**: Must be a signer/owner of the Gnosis Safe multisig
2. **Required Threshold**: (For execution) Sufficient signers must be available to meet the Safe's signature threshold
3. **Network Access**: Connection to the Sepolia testnet
4. **Function Parameters**: All required parameters for the `attachAggchainToAL` function
5. **Coordination**: (For proposers) Coordination with Safe signers for transaction review and approval

### For Genesis Generation

Depending on your client type, you'll need:

**For CDK-Erigon Networks:**
- Go installed
- Access to an L1 network RPC endpoint
- Knowledge of the AgglayerManager and rollup details

**For Vanilla Client Networks:**
- Node.js and npm installed
- Access to an L1 network RPC endpoint
- Knowledge of your rollup details

## Function Parameters

The `attachAggchainToAL` function requires the following parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `rollupTypeID` | `uint32` | The unique identifier of the rollup type to be attached |
| `chainID` | `uint64` | The chain ID of the chain |
| `initializeBytesAggchain` | `bytes` | The AggchainManager EOA address that will initialize the rollup in a 2nd stage |

> [!NOTE]
> Please verify the exact function signature and parameters on Etherscan before execution:
> - **Bali**: [Etherscan Link](https://sepolia.etherscan.io/address/0xE2EF6215aDc132Df6913C8DD16487aBF118d1764#writeProxyContract#F4)
> - **Cardona**: [Etherscan Link](https://sepolia.etherscan.io/address/0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff#writeProxyContract#F4)

## Part 1: Chain Integration Process

### For Proposers Only (e.g., Polygon IPs)

#### 1. Access the Gnosis Safe Interface

1. Navigate to [Gnosis Safe App](https://app.safe.global/)
2. Connect your wallet and access the appropriate Safe
3. Ensure you're connected to the **Sepolia testnet**

#### 2. Propose the Transaction

1. In the Safe interface, click on **"New Transaction"**
2. Select **"Contract Interaction"**
3. Enter the appropriate contract address:
   - **Bali**: `0xE2EF6215aDc132Df6913C8DD16487aBF118d1764`
   - **Cardona**: `0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff`
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
   - **initializeBytesAggchain**: Enter the AggchainManager EOA address as hex-encoded bytes

> [!IMPORTANT]
> When selecting the `rollupTypeID`, visit Etherscan to find the number of rollupTypeIds, get details about each rollupTypeId, and access each rollupTypeId's `consensusImplementation` contract. Use the appropriate contract address:
> - **Bali**: [Number of rollupTypeIds](https://sepolia.etherscan.io/address/0xE2EF6215aDc132Df6913C8DD16487aBF118d1764#readProxyContract#F28) | [rollupTypeId details](https://sepolia.etherscan.io/address/0xE2EF6215aDc132Df6913C8DD16487aBF118d1764#readProxyContract#F29)
> - **Cardona**: [Number of rollupTypeIds](https://sepolia.etherscan.io/address/0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff#readProxyContract#F28) | [rollupTypeId details](https://sepolia.etherscan.io/address/0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff#readProxyContract#F29)
>
> Review the rollupTypeIds from the latest one going backwards until you find the appropriate one based on these criteria:
> - For **AggchainECDSAMultisig** rollups (zkEVM/Validium/PP rollups): Select a rollupTypeId whose consensusImplementation is defined with `AGGCHAIN_TYPE = 0x0000`
> - For **AggchainFEP** rollups: Select a rollupTypeId whose consensusImplementation is defined with `AGGCHAIN_TYPE = 0x0001`

> [!IMPORTANT]
> The `initializeBytesAggchain` parameter now contains a single field: the **AggchainManager EOA address**. This account will be used to initialize the rollup in a 2nd stage operation that the IP will perform themselves.
>
> You can encode the address using:

<details>
<summary>Using JavaScript</summary>

```javascript
const ethers = require('ethers');

/**
 * Function to encode the AggchainManager address
 * @param {String} aggchainManagerAddress The EOA address that will initialize the rollup
 * @returns {String} encoded value in hexadecimal string
 */
function encodeAggchainManagerAddress(aggchainManagerAddress) {
    return ethers.AbiCoder.defaultAbiCoder().encode(
        ['address'],
        [aggchainManagerAddress]
    );
}
```

</details>

<details>
<summary>Using cast (Foundry)</summary>

```shell
$ cast abi-encode "encode(address)" <aggchainManagerAddress>
```

</details>

## Post-Attachment: Rollup Initialization

After successful chain attachment, the IP must initialize the rollup using the AggchainManager account. The IP will call the `initialize` function on the **Rollup contract**. The initialization function varies depending on the rollup type:

### For AggchainECDSAMultisig Rollups (zkEVM/Validium/PP rollups)

```solidity
struct SignerInfo {
   address addr;
   string url;
}

function initialize(
    address _admin,
    address _trustedSequencer,
    address _gasTokenAddress,
    string memory _trustedSequencerURL,
    string memory _networkName,
    bool _useDefaultSigners,
    SignerInfo[] memory _signersToAdd,
    uint256 _newThreshold
) external onlyAggchainManager
```

**Default Configuration**: IPs should set the trusted sequencer as the only signer by setting:
- `_useDefaultSigners`: `false`
- `_signersToAdd`: array containing the trusted sequencer address `[(trustedSequencerAddress, trustedSequencerURL)]`
- `_newThreshold`: `1`

### For AggchainFEP Rollups

```solidity
/// @notice Initialize function for fresh deployment
/// @custom:security Initializes all contracts including PolygonConsensusBase
/// @param _initParams The initialization parameters for FEP
///    struct InitParams {
///          uint256 l2BlockTime;
///          bytes32 rollupConfigHash;
///          bytes32 startingOutputRoot;
///          uint256 startingBlockNumber;
///          uint256 startingTimestamp;
///          uint256 submissionInterval;
///          address optimisticModeManager;
///          bytes32 aggregationVkey;
///          bytes32 rangeVkeyCommitment;
///       }
/// @param _signersToAdd Array of signers to add to the multisig
/// @param _newThreshold New threshold for multisig operations
/// @param _useDefaultVkeys Whether to use default verification keys from gateway
/// @param _useDefaultSigners Whether to use default signers from gateway
/// @param _initOwnedAggchainVKey The owned aggchain verification key
/// @param _initAggchainVKeySelector The aggchain verification key selector
/// @param _admin The admin address
/// @param _trustedSequencer The trusted sequencer address
/// @param _gasTokenAddress The gas token address
/// @param _trustedSequencerURL The trusted sequencer URL
/// @param _networkName The network name
function initialize(
    InitParams memory _initParams,
    SignerInfo[] memory _signersToAdd,
    uint256 _newThreshold,
    bool _useDefaultVkeys,
    bool _useDefaultSigners,
    bytes32 _initOwnedAggchainVKey,
    bytes4 _initAggchainVKeySelector,
    address _admin,
    address _trustedSequencer,
    address _gasTokenAddress,
    string memory _trustedSequencerURL,
    string memory _networkName
) external onlyAggchainManager
```

#### 4. Submit Proposal

1. **Review all parameters carefully** - incorrect values could cause the transaction to fail or have unintended consequences
2. Check the estimated gas fees
3. Click **"Review"** to see the transaction summary
4. If everything looks correct, click **"Submit"** to create the transaction proposal
5. **Share the transaction link** with the Safe signers for review and approval

### For Signers/Owners

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

## Part 2: Network Genesis File Generation

After successfully attaching the chain to the Agglayer and verifying the transaction, the next step is to generate the network genesis file for your chain. The procedure differs depending on the client that the network is based on.

### For CDK-Erigon Networks

If your network runs with `cdk-erigon`, use the [cdk-contracts-tooling](https://github.com/0xPolygon/cdk-contracts-tooling) project to generate the genesis file.

#### Steps

1. **Clone and setup the cdk-contracts-tooling repository:**
   ```bash
   git clone https://github.com/0xPolygon/cdk-contracts-tooling.git
   cd cdk-contracts-tooling
   ```

2. **Configure RPCs and wallets:**
   ```bash
   # Copy and configure RPC endpoints
   cp rpcs.example.toml rpcs.toml
   # Edit rpcs.toml to include your L1 network RPC configuration
   ```

3. **Import the AgglayerManager:**
   ```bash
   go run ./cmd import-rm -l1 <L1_NETWORK> -addr <AGGLAYER_MANAGER_ADDRESS> -alias <ALIAS>
   ```

   Example:
   ```bash
   go run ./cmd import-rm -l1 sepolia -addr 0x32d33D5137a7cFFb54c5Bf8371172bcEc5f310ff -alias cardona
   ```

4. **Import the rollup (if not automatically imported):**
   ```bash
   go run ./cmd import-r -l1 <L1_NETWORK> -rm <RM_ALIAS> -r <ROLLUP_NAME> -chainid <CHAIN_ID>
   ```

   Example:
   ```bash
   go run ./cmd import-r -l1 sepolia -rm cardona -r MyChain -chainid 1234567
   ```

5. **Generate the genesis file:**
   ```bash
   go run ./cmd genesis -l1 <L1_NETWORK> -rm <RM_ALIAS> -r <ROLLUP_ALIAS> -output <OUTPUT_FILE>
   ```

   Example:
   ```bash
   go run ./cmd genesis -l1 sepolia -rm cardona -r MyChain -output MyChain-genesis.json
   ```

6. **Generate bridge service configuration (optional):**
   ```bash
   go run ./cmd bridge -l1 <L1_NETWORK> -rm <RM_ALIAS> -r <ROLLUP_ALIAS> -output <OUTPUT_FILE>
   ```

7. **Generate combined json file (optional, contains various information about the network):**
   ```bash
   go run ./cmd import-combined-json -l1 <L1_NETWORK> -rm <RM_ALIAS> -r <ROLLUP_ALIAS>
   ```

### For Vanilla Client Networks

If your network runs with vanilla clients (like standard geth, op-geth, etc.), use the [createSovereignGenesis tool](https://github.com/agglayer/agglayer-contracts/tree/v11.0.0-rc.3/tools/createSovereignGenesis) from the agglayer-contracts repository.

#### Steps

1. **Clone the agglayer-contracts repository:**
   ```bash
   git clone https://github.com/agglayer/agglayer-contracts.git
   cd agglayer-contracts
   git checkout v11.0.0-rc.3
   ```

2. **Navigate to the createSovereignGenesis tool:**
   ```bash
   cd tools/createSovereignGenesis
   ```

3. **Install dependencies:**
   ```bash
   npm install
   ```

4. **Configure the tool:**
   - Follow the tool's documentation to configure the necessary parameters
   - Provide your L1 RPC endpoint and AgglayerManager contract details
   - Specify your chain configuration parameters

5. **Generate the genesis file:**
   ```bash
   # Follow the specific commands provided in the tool's README
   # The exact command will depend on the tool's interface
   npm run generate-genesis -- --config <your-config-file>
   ```

> [!NOTE]
> Refer to the [createSovereignGenesis documentation](https://github.com/agglayer/agglayer-contracts/tree/v11.0.0-rc.3/tools/createSovereignGenesis) for detailed usage instructions and configuration options.

### Important Considerations

- **Client Compatibility**: Ensure you're using the correct tool for your client type
- **Network Configuration**: Verify that all network parameters match your chain's configuration
- **Genesis Validation**: Always validate the generated genesis file before deploying
- **Backup**: Keep backups of your genesis files and configuration
- **Documentation**: Document the parameters and tools used for future reference

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
