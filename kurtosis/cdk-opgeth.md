# Quickstart: Run CDK-opgeth locally with Kurtosis

Use this guide to deploy a local devnet instance of cdk-opgeth using Kurtosis. This includes a local L1 + L2 environment with Agglayer components and OP Stack infrastructure.

> This guide has been updated on **2025-12-03**. You can always refer to the [kurtosis-cdk documentation](https://0xpolygon.github.io/kurtosis-cdk/) for an up-to-date version.

## 1. Installation

To get started, you will need to install a few required tools to deploy the local environment:

- [kurtosis](https://github.com/kurtosis-tech/kurtosis)
- [docker](https://docs.docker.com/)

You will also need the following tools to interact with the deployed environment:

- [foundry toolchain](https://github.com/foundry-rs/foundry) - Ethereum development toolkit
- [polycli](https://github.com/0xPolygon/polygon-cli) - swiss army knife of blockchain tools developed by engineers at Polygon

## 2. Deploy the devnet

Use the command below to deploy the environment:

```bash
kurtosis run \
  --enclave cdk \
  --args-file https://raw.githubusercontent.com/0xPolygon/kurtosis-cdk/refs/tags/v0.5.10/.github/tests/op-succinct/mock-prover.yml \
  github.com/0xPolygon/kurtosis-cdk@v0.5.10
```

This will:

- Deploy an L1 devnet (Ethereum-like chain)
- Deploy [agglayer contracts](https://github.com/agglayer/agglayer-contracts)
- Deploy the [agglayer](https://github.com/agglayer/agglayer) and its mock [prover](https://github.com/agglayer/provers), enabling trustless cross-chain token transfers and message-passing, as well as more complex operations between L2 chains, secured by zk proofs.
- Deploy an L2 devnet (OP stack), enhanced with the [op-succinct-proposer](https://github.com/agglayer/op-succinct) to support zk proof generation and [aggkit](https://github.com/agglayer/aggkit) for seamless agglayer connectivity.
- Deploy the legacy [zkevm bridge](https://github.com/0xPolygonHermez/zkevm-bridge-service) to facilitate asset bridging between L1 and L2 chains.

You should see output similar to this:

```bash
========================================== User Services ==========================================
UUID           Name                                             Ports                                                Status
c740c0ec9191   aggkit-001                                       pprof: 6060/tcp -> http://127.0.0.1:32868            RUNNING
                                                                rpc: 5576/tcp -> http://127.0.0.1:32867              
91cc3337d94e   aggkit-001-bridge                                pprof: 6060/tcp -> http://127.0.0.1:32871            RUNNING
                                                                rest: 5577/tcp -> http://127.0.0.1:32870             
                                                                rpc: 5576/tcp -> http://127.0.0.1:32869              
7b7644053ea0   aggkit-prover-001                                grpc: 4446/tcp -> grpc://127.0.0.1:32865             RUNNING
                                                                metrics: 9093/tcp -> http://127.0.0.1:32866          
fed320909771   agglayer                                         aglr-admin: 4446/tcp -> http://127.0.0.1:32861       RUNNING
                                                                aglr-grpc: 4443/tcp -> grpc://127.0.0.1:32859        
                                                                aglr-readrpc: 4444/tcp -> http://127.0.0.1:32860     
                                                                prometheus: 9092/tcp -> http://127.0.0.1:32862       
4c9328c86c71   agglayer-dashboard                               dashboard: 8000/tcp -> http://127.0.0.1:32875        RUNNING
fa1575a28daa   agglayer-prover                                  api: 4445/tcp -> grpc://127.0.0.1:32857              RUNNING
                                                                prometheus: 9093/tcp -> http://127.0.0.1:32858       
1c911fd4177f   agglogger                                        <none>                                               RUNNING
f6710c6fb2c5   bridge-spammer-001                               <none>                                               RUNNING
5dbc41a82b81   cl-1-lighthouse-geth                             http: 4000/tcp -> http://127.0.0.1:32843             RUNNING
                                                                metrics: 5054/tcp -> http://127.0.0.1:32844          
                                                                quic-discovery: 9001/udp                             
                                                                tcp-discovery: 9000/tcp -> 127.0.0.1:32845           
                                                                udp-discovery: 9000/udp                              
f52bdd66aca8   contracts-001                                    http: 8080/tcp -> http://127.0.0.1:32847             RUNNING
29e27d7a09e3   el-1-geth-lighthouse                             engine-rpc: 8551/tcp -> 127.0.0.1:32840              RUNNING
                                                                metrics: 9001/tcp -> http://127.0.0.1:32841          
                                                                rpc: 8545/tcp -> 127.0.0.1:32838                     
                                                                tcp-discovery: 30303/tcp -> 127.0.0.1:32842          
                                                                udp-discovery: 30303/udp                             
                                                                ws: 8546/tcp -> 127.0.0.1:32839                      
56cde591b111   op-batcher-001                                   http: 8548/tcp -> http://127.0.0.1:32854             RUNNING
aeead0d07471   op-cl-1-op-node-op-geth-001                      rpc: 8547/tcp -> http://127.0.0.1:32852              RUNNING
                                                                tcp-discovery: 9003/tcp -> http://127.0.0.1:32853    
                                                                udp-discovery: 9003/udp                              
43b3207d5874   op-el-1-op-geth-op-node-001                      engine-rpc: 8551/tcp -> http://127.0.0.1:32850       RUNNING
                                                                rpc: 8545/tcp -> http://127.0.0.1:32848              
                                                                tcp-discovery: 30303/tcp -> http://127.0.0.1:32851   
                                                                udp-discovery: 30303/udp                             
                                                                ws: 8546/tcp -> ws://127.0.0.1:32849                 
e56d03628f4a   op-succinct-proposer-001                         grpc: 50051/tcp -> grpc://127.0.0.1:32864            RUNNING
                                                                prometheus: 8080/tcp -> http://127.0.0.1:32863       
8d3c8a3e727e   postgres-001                                     postgres: 5432/tcp -> postgresql://127.0.0.1:32856   RUNNING
5d7857606076   proxyd-001                                       http: 8080/tcp -> http://127.0.0.1:32855             RUNNING
0262b8847105   test-runner                                      <none>                                               RUNNING
c03b5047d216   validator-key-generation-cl-validator-keystore   <none>                                               RUNNING
2766f5b04984   vc-1-geth-lighthouse                             metrics: 8080/tcp -> http://127.0.0.1:32846          RUNNING
9f9d9ed82ca9   zkevm-bridge-service-001                         grpc: 9090/tcp -> grpc://127.0.0.1:32874             RUNNING
                                                                prometheus: 8090/tcp -> http://127.0.0.1:32873       
                                                                rpc: 8080/tcp -> http://127.0.0.1:32872 
```

Retrieve the L1 and L2 RPC urls - it will be convenient for the next steps:

```bash
export L1_RPC_URL=http://$(kurtosis port print cdk el-1-geth-lighthouse rpc)
export L2_RPC_URL=$(kurtosis port print cdk op-el-1-op-geth-op-node-001 rpc)
```

## 3. Bridge funds from L1 to L2

For the purpose of this example, we will generate a new wallet with `cast`:

```bash
cast wallet new --json
```

```json
[
  {
    "address": "0x4836Ba780a31902EF78a7D0Ed53873065c1018CC",
    "private_key": "0x9723674a48efa7ee9c73157aa72cae9e409efa1bc729a7a3691b002876c1da51"
  }
]
```

We will use the above address `0x4836Ba780a31902EF78a7D0Ed53873065c1018CC` as the recipient on L2.

Use `polycli` to bridge assets:

```bash
polycli ulxly bridge asset \
  --rpc-url $L1_RPC_URL \
  --bridge-address $(kurtosis service exec cdk contracts-001 "jq -r '.polygonZkEVMBridgeAddress' /opt/zkevm/combined.json") \
  --private-key 0x12d7de8621a77640c9241b2595ba78ce443d05e94090365ab3bb5e19df82c625 \
  --destination-address 0x4836Ba780a31902EF78a7D0Ed53873065c1018CC \
  --destination-network 1 \
  --value $(cast to-wei 1)
```

Let's break down this command:
- `polycli ulxly bridge asset`: initiates the bridge
- `--rpc-url`: L1 RPC url
- `--bridge-address`: reads the bridge contract address from `combined.json`, located in the `contracts-001` service
- `--private-key`: use the admin private key to send the transaction
- `--destination-address`: L2 recipient
- `--destination-network`: target network ID (`0` for L1 and `1` for L2)
- `--value`: amount to bridge in wei - in this case, one ether

If successful, you’ll see a transaction confirmation and the bridged amount will appear in the destination address on L2.

```bash
The command was successfully executed and returned '0'.
12:51PM INF bridgeTxn: 0xb3fd667f3e258dd2650f9325acc5512126c3751f17271a0ac1d4225be92d4588
12:51PM INF transaction successful txHash=0xb3fd667f3e258dd2650f9325acc5512126c3751f17271a0ac1d4225be92d4588
12:51PM INF Bridge deposit count parsed from logs depositCount=15
```

Check balance on L2:

```bash
cast balance --rpc-url $L2_RPC_URL --ether 0x4836Ba780a31902EF78a7D0Ed53873065c1018CC
```

You should see `1.000000000000000000` as the output.

## 4. Send a transaction on L2

Use `cast` to send a transaction:

```bash
cast send \
  --rpc-url $L2_RPC_URL \
  --private-key 0x9723674a48efa7ee9c73157aa72cae9e409efa1bc729a7a3691b002876c1da51 \
  0x4836Ba780a31902EF78a7D0Ed53873065c1018CC \
  $(echo -n 'data:,Hello Agglayer!' | xxd -p)
```

Note that we use the private key generated in step 3.

This command sends a transaction with `Hello Agglayer!` embedded in the calldata.

You should see output similar to this:

```bash
blockHash            0x17c2840507a4dd4518ef5d11de6b6f6c6f8b0c5d4101b546cb387bd4c94f83f7
blockNumber          1368
contractAddress      
cumulativeGasUsed    202020
effectiveGasPrice    4277369
from                 0x4836Ba780a31902EF78a7D0Ed53873065c1018CC
gasUsed              21840
logs                 []
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root                 
status               1 (success)
transactionHash      0xb8c2e7ac5f6750525e78249d9b594336be1504f14f7a055333f4acb00432595b
transactionIndex     2
type                 2
blobGasPrice         
blobGasUsed          
to                   0x4836Ba780a31902EF78a7D0Ed53873065c1018CC
l1BaseFeeScalar      1368
l1BlobBaseFee        1
l1BlobBaseFeeScalar  810949
l1Fee                96
l1GasPrice           7
l1GasUsed            1600
```

Retrieve the tx hash from the output and check its status:

```bash
cast tx --rpc-url $L2_RPC_URL 0xb8c2e7ac5f6750525e78249d9b594336be1504f14f7a055333f4acb00432595b input | xxd -r -p
```

It should output `data:,Hello Agglayer!`.

## 5. Bridge funds back from L2 to L1

Use `polycli` to bridge assets:

```bash
polycli ulxly bridge asset \
  --rpc-url $L2_RPC_URL \
  --bridge-address $(kurtosis service exec cdk contracts-001 "jq -r '.polygonZkEVML2BridgeAddress' /opt/zkevm/combined.json") \
  --private-key 0x9723674a48efa7ee9c73157aa72cae9e409efa1bc729a7a3691b002876c1da51 \
  --destination-address 0x4836Ba780a31902EF78a7D0Ed53873065c1018CC \
  --destination-network 0 \
  --value $(cast to-wei 0.5)
```

Note that we use the private key generated in step 3 and the destination network is now `0` (L1).

You should see a transaction confirmation similar to this:

```bash
The command was successfully executed and returned '0'.
1:00PM INF bridgeTxn: 0xab6d1318f0be01a7a4fc1f38c6cd8e1bf51d1c78758751e7b0ef86415d5d005d
1:00PM INF transaction successful txHash=0xab6d1318f0be01a7a4fc1f38c6cd8e1bf51d1c78758751e7b0ef86415d5d005d
1:00PM INF Bridge deposit count parsed from logs depositCount=20
```

Attempt to claim the bridged amount on L1:

```bash
polycli ulxly claim asset \
  --rpc-url $L1_RPC_URL \
  --bridge-address $(kurtosis service exec cdk contracts-001 "jq -r '.polygonZkEVMBridgeAddress' /opt/zkevm/combined.json") \
  --private-key 0x12d7de8621a77640c9241b2595ba78ce443d05e94090365ab3bb5e19df82c625 \
  --deposit-count 20 \
  --deposit-network 1 \
  --destination-address 0x4836Ba780a31902EF78a7D0Ed53873065c1018CC \
  --bridge-service-url $(kurtosis port print cdk zkevm-bridge-service-001 rpc) \
  --wait "10m"
```

After some time, the transaction should be claimable on L1. You should see output similar to this:

```bash
The command was successfully executed and returned '0'.
1:55PM INF The deposit is ready to be claimed
1:55PM INF claimTxn: 0xad35dbfe959a89e527dbab90e3cab525efe0469333bb78a28fa76a84a307a38c
1:55PM INF transaction successful txHash=0xad35dbfe959a89e527dbab90e3cab525efe0469333bb78a28fa76a84a307a38c
```

You can check the balance on L1:

```bash
cast balance --rpc-url $L1_RPC_URL --ether 0x4836Ba780a31902EF78a7D0Ed53873065c1018CC
```

You should see `0.500000000000000000` as the output.

## 6. Clean up

Once you are done experimenting, you can remove the environment with:

```bash
kurtosis enclave rm --force cdk
```
