# Getting Started

This runbook aims at describing the complete Agglayer stack and how L2 interact with it.

The stack is divided into what Polygon Labs runs (Agglayer) and what each L2 runs (Aggkit).

## Agglayer

Polygon Labs runs the Agglayer which consists in these two components:
- agglayer-node
- agglayer-prover

### agglayer-node

agglayer-node syncs/pushes data from/to L1 (e.g., Pessimistic Proofs—aka PP). It also exposes a JSON-RPC/gRPC API used by L2 to submit certificates.

Chains connected to Agglayer can be of various types, defined by `CONSENSUS_TYPE`. The `CONSENSUS_TYPE` defines how the PP will verify the state transition pushed by the chain:
- `0`: Certificate ECDSA signature verification
- `1`: Aggchain proof verification

Aggchain proof can also be of various types, defined by `AGGCHAIN_TYPE`:
- `1`: OP-FEP proof computed by op-succinct

When it receives a certificate, agglayer-node verifies it is valid with its own agglayer state, and executes the PP program to generate the witnesses (the output of the program execution).

Additionally, agglayer-node retrieves the aggchain proof vkey from L1 via the Agglayer Gateway contract, and sends it to agglayer-prover for it to compute the PP.

### agglayer-prover

agglayer-prover computes the PP by running the PP program on the SP1 network.

## Aggkit

Each L2 runs Aggkit, which consists in these two components:
- aggkit (aggsender, aggoracle, bridge, etc.)
- aggkit-prover

### aggkit

aggkit runs with a configurable list of internal features:
- aggsender: sends certificates to agglayer-node via JSON-RPC (pre-v0.3) / gRPC (post-v0.3)
- aggoracle: reads the GER from L1 and sets it on L2
- bridge: unified bridge service

### aggkit-prover

aggkit-prover runs for `CONSENSUS_TYPE = 1` only. It computes the aggchain proof by running the aggchain-proof program on the SP1 network.
