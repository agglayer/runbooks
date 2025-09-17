# Running Aggsender Validator

This runbook provides guidance for Implementation Providers (IPs) on deploying and configuring the `aggsender-validator` component using AggKit. The aggsender-validator is responsible for validating and signing certificates in the Aggsender committee.

## Overview

The `aggsender-validator` component serves two primary deployment scenarios for Implementation Providers:

1. **Integrated Deployment**: IPs running their own L2 chain include aggsender-validator as part of their standard AggKit deployment (with the aggsender)
2. **Standalone Deployment**: IPs run an isolated aggsender-validator instance to participate in the committee for L2 chains operated by other IPs

Both scenarios contribute to the decentralized validation of certificates submitted to the Agglayer.

## Deployment Scenarios

### Scenario 1: Integrated Deployment (Own L2 Chain)

**When to use**: You are the Implementation Provider operating your own L2 chain and want to participate in certificate validation.

**Components to run**:
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,bridge
```

> [!info]
> **If your L2 stack is the OP Stack:**
> You must also add the `aggoracle` component to your deployment.
> Example:
> ```bash
> aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender,bridge,aggoracle
> ```


**Key characteristics**:
- Runs alongside your existing L2 infrastructure
- Validates certificates for your own chain and others in the committee
- Requires full L2 chain access and configuration
- Part of your standard AggKit deployment

### Scenario 2: Standalone Deployment (Committee Participation)

**When to use**: You want to participate in the aggsender committee for L2 chains operated by other Implementation Providers.

**Components to run**:
```bash
aggkit run --cfg=/etc/aggkit/config.toml --components=aggsender-validator
```

**Key characteristics**:
- Isolated deployment focused solely on validation
- Does not require running your own L2 chain
- Participates in multi-signature validation for other chains
- Minimal infrastructure requirements

## Configuration Requirements

### Core Configuration Sections

All aggsender-validator deployments require configuration of the validator-specific section:

<details>
<summary>Validator configuration section</summary>

```toml
# Validator configuration
[Validator]
# Enable RPC server for validator
EnableRPC = false
# Signer configuration (uses same format as AggsenderPrivateKey)
Signer = {Method = "local", Path = "/etc/aggkit/validator.keystore", Password = "***"}
# Maximum certificate size (inherits from AggSender if not specified)
MaxCertSize = 8388608
# Maximum L2 block number to process
MaxL2BlockNumber = 0
# Delay between retries when processing fails
DelayBetweenRetries = "30s"

# Validator server configuration
[Validator.ServerConfig]
Host = "0.0.0.0"
Port = 5578
EnableReflection = true
MaxDecodingMessageSize = 26214400  # 25Mb

# LER (Local Exit Root) querier configuration
[Validator.LerQuerierConfig]
RollupManagerAddr = "0x0000000000000000000000000000000000000000"
RollupCreationBlockL1 = 0

# Pessimistic Proof configuration
[Validator.PPConfig]
RequireOneBridgeInPPCertificate = false

# Agglayer client configuration for validator
[Validator.AgglayerClient]
Cached = true
[Validator.AgglayerClient.ConfigurationCache]
TTL = "5m"
Capacity = 100
[Validator.AgglayerClient.GRPC]
URL = "https://agglayer-dev.polygon.technology"
MinConnectTimeout = "5s"
RequestTimeout = "300s"
UseTLS = false
[Validator.AgglayerClient.GRPC.Retry]
InitialBackoff = "1s"
MaxBackoff = "10s"
BackoffMultiplier = 2.0
MaxAttempts = 20
```

</details>

### Scenario-Specific Configuration

#### Integrated Deployment

For IPs running their own L2 chain, the aggsender-validator runs alongside the standard AggKit components. The core validator configuration above is required, plus one additional configuration change to connect the aggsender to the local validator:

<details>
<summary>Additional configuration for integrated deployment</summary>

```toml
# Configure AggSender to communicate with local validator
[AggSender.ValidatorClient]
URL = "http://localhost:5578"  # Points to local validator server
MinConnectTimeout = "5s"
RequestTimeout = "30s"
UseTLS = false
```

</details>

The aggsender-validator will use the existing AggKit configuration for:
- L1/L2 connectivity
- Contract addresses
- Logging settings
- Other standard AggKit parameters

Refer to the main [L2 trusted environment runbook](./run-l2-trusted-environment.md) for complete AggKit configuration.

#### Standalone Deployment

For standalone validator instances, you need the core validator configuration plus minimal AggKit base configuration for connectivity.

Refer to the main [L2 trusted environment runbook](./run-l2-trusted-environment.md) for complete AggKit configuration.

## Key Management and Security

### Private Key Requirements

**Integrated Deployment**:
- Aggsender private key (for certificate generation)
- Validator private key (for certificate validation/signing)

**Standalone Deployment**:
- Validator private key only (for certificate validation/signing)

## Committee Participation

### Committee Structure

The aggsender committee operates on a multi-signature model:

- **Committee Size**: Configurable number of validators (typically 5-7)
- **Threshold**: Minimum signatures required for certificate approval (typically 3-5)
- **Participation**: Each validator independently validates and signs certificates

### Validator Responsibilities

1. **Certificate Validation**: Verify certificate integrity and correctness
2. **Signature Generation**: Sign valid certificates with validator private key
3. **Committee Coordination**: Participate in threshold signature collection
