# Rollup Network Creation

This document describes the initial step of requesting and receiving rollup creation artifacts from Polygon Labs. This is the first step in the deployment process and must be completed before proceeding with genesis generation and network deployment.

## TL;DR

1. The IP proposes the `attachAggchainToAL()` transaction on Gnosis Safe
2. Polygon Labs executes the transaction
3. Polygon Labs shares `combined.json` with core deployment details

## Step 1: Propose the `attachAggchainToAL()` Transaction

The **Implementation Partner (IP)** proposes the `attachAggchainToAL()` transaction on the Gnosis Safe UI.

Follow the instructions in the [Chain Integration Process](../../operations/initiate-chain-integration-agglayer.md#part-1-chain-integration-process) runbook.

## Step 2: Transaction Execution by Polygon Labs

Once the transaction is proposed:

- Polygon Labs reviews and executes the transaction.
- A transaction is recorded on-chain (e.g., [example transaction](https://sepolia.etherscan.io/tx/0x9ff8f3556bc6b1f8b9b71fffc9e9434fdf0ae7d0d525b75611cce452c9cdb305)).

## Step 3: Receive Deployment Artifacts

After execution, Polygon Labs will share the following file:

- `combined.json` - Core deployment details

> 💡 **Important:** Secure these files. They are required for further deployment steps.
