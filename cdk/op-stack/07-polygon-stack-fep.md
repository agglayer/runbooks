# Polygon Stack FEP Deployment

This document describes how to run the components required for Polygon Stack FEP (Full Execution Proof) deployment.

> **Note:** Docker images have been tested on AMD64 (linux/amd64). Other architectures are not guaranteed to work.

## Environment Variables

```shell
# See Component Versions table in README.md for current values
export aggkit_prover_version="<aggkit_prover_version>"
export op_succinct_version="<op_succinct_version>"
```

## Aggkit Prover

### Prerequisites

- Access to L1 and L2 RPC endpoints

### Configuration

> **Source of Truth**: For complete configuration documentation, see the [example configuration snapshot](https://github.com/agglayer/provers/blob/main/crates/aggkit-prover-config/tests/snapshots/validate_deserialize__prover_grpc_max_decoding_message_size.snap).

The Aggkit Prover uses a TOML configuration file. Create a configuration file at `/etc/aggkit/aggkit-prover-config.toml`. See the example configuration snapshot for reference.

### Running the Container

Run the Aggkit Prover using Docker with the configuration file mounted:

```shell
docker run --rm -it \
  -v /path/to/aggkit-prover-config.toml:/etc/aggkit/aggkit-prover-config.toml \
  -v /path/to/op-stack/genesis.json:/etc/aggkit/genesis.json \
  ghcr.io/agglayer/aggkit-prover:${aggkit_prover_version} \
  /usr/local/bin/aggkit-prover run --config-path /etc/aggkit/aggkit-prover-config.toml
```

### Additional Resources

- **GitHub**: [agglayer/provers](https://github.com/agglayer/provers)

## OP Succinct Proposer

> **Source of Truth**: For complete documentation, see the [OP Succinct Proposer Documentation](https://github.com/agglayer/op-succinct/blob/main/book/validity/proposer.md).

### Prerequisites

- A dedicated PostgreSQL database (must be set up separately)
- Access to L1 and L2 RPC endpoints

### Environment Variables

> **Source of Truth**: For complete environment variable documentation, see the [Environment Setup](https://github.com/agglayer/op-succinct/blob/main/book/validity/proposer.md#environment-setup) section.

### Running the Container

Make sure to include *all* of the required environment variables in the `.env` file. See the documentation for the complete list of required and optional environment variables.

```shell
docker run --rm -it \
  --env-file .env \
  ghcr.io/agglayer/op-succinct/op-succinct-agglayer:${op_succinct_version}
```

### Additional Resources
- **GitHub**: [agglayer/op-succinct](https://github.com/agglayer/op-succinct)
