# PP to FEP upgrade

> ![warning] Only AMD64 (linux/amd64) Docker images have been tested and are guaranteed to be supported. Other architectures are not officially supported.

## Requirements
- All paths in this guide are relative to the `upgrade` directory:
```
mkdir upgrade
```

```
cd upgrade
```

- Polygon Labs provides a `combined.json` file as part of the rollup creation process. Save a copy of this file in the upgrade directory.

- At this point, your folder layout should look like this:

```
upgrade % tree .
.
└── combined.json

1 directory, 1 file
```

## Environment
```
cat << EOF > .env
export network_name="" # replace by the network name. Can be retrieved by invoking the networkName function of the rollup smart contract

# services
export agglayer_url=http:// # readrpc agglayer url (e.g.: https://agglayer-dev.polygon.technology for Bali)
export l2_rpc_url=http:// # op-geth url
export l2_node_url=http:// # op-node url
export l1_rpc_url=http:// # l1 rpc url

# wallet
export admin_address=0x # replace by the rollup admin address

# rollup
export new_rollup_type_id=0 # to be provided by Polygon labs

# fetched from the combined.json file
export l2_chain_id=$(cat combined.json | jq -r .l2ChainID)
export rollup_id=$(cat combined.json | jq -r .rollupID)

# fetched from the docker images
export aggchain_vkey=$(
  docker run --rm -it \
    --platform linux/amd64 \
    ghcr.io/agglayer/aggkit-prover:1.1.2 \
    /usr/local/bin/aggkit-prover vkey
)

export aggchain_vkey_selector=$(
  docker run --rm -it \
    --platform linux/amd64 \
    ghcr.io/agglayer/aggkit-prover:1.1.2 \
    /usr/local/bin/aggkit-prover vkey-selector
)
EOF
```

```
source .env
```

## Upgrade

### Stop the components

- stop the aggkit

- stop sequencing

```
cast rpc --rpc-url "$l2_node_url" admin_stopSequencer
```

### Rollup Update

```
git clone https://github.com/agglayer/agglayer-contracts
```

```
cd agglayer-contracts
```

```
git checkout v10.1.0-rc.8
```

```
upgrade_data=$(cast calldata "initAggchainManager(address)" ${admin_address})
```

```
jq \
  --arg upgrade_data "$upgrade_data" \
  --arg new_rollup_type_id "$new_rollup_type_id" \
  --slurpfile c ../combined.json \
  '
  .polygonRollupManagerAddress = $c[0].polygonRollupManagerAddress |
  .timelockDelay = "@@replace" |
  .deployerPvtKey = "@@replace" |
  .rollups[0].rollupAddress = $c[0].rollupAddress |
  .rollups[0].newRollupTypeID = $new_rollup_type_id |
  .rollups[0].upgradeData = $upgrade_data
  ' ./tools/updateRollup/updateRollup.json.example \
    > ./tools/updateRollup/updateRollup.json
```

> **timelockDelay**: the duration of the timelock. It's advisable to use a short delay for devnets (e.g., 60) for convenience

> **deployerPvtKey**: the private key used for executing smart contract operations

Review the content of `./tools/updateRollup/updateRollup.json` and update the placeholders marked with `@@replace` with the appropriate values. 

```
npm i
npx hardhat run ./tools/updateRollup/updateRollup.ts --network sepolia
```

Navigate back to the parent directory
```
cd ..
```

### Rollup Initialization
> ⚠️ This is likely the most critical step — initializing the rollup with incorrect values can lead to an irrecoverably broken network state.
#### Fetch the L2 block containing the most recent settled certificate
We must ensure that there are no pending certificates — the following command should return `null`
```
cast rpc interop_getLatestPendingCertificateHeader $rollup_id --rpc-url $agglayer_url | jq .
```

Retrieve the L2 block that includes the most recently settled certificate
```
latest_settled_l2_block=$(
  cast rpc --rpc-url $agglayer_url interop_getLatestSettledCertificateHeader $rollup_id |
  jq -r '.metadata' |
  perl -e '
    $_ = <>;
    s/^\s+|\s+$//g;
    s/^0x//;
    $_ = pack("H*", $_);
    my ($v, $f, $o, $c) = unpack("C Q> L> L>", $_);
    printf "{\"v\":%d,\"f\":%d,\"o\":%d,\"c\":%d}\n", $v, $f, $o, $c;
  ' |
  jq '.f + .o'
)
```
> 💡 It's very **important** to get the correct number

Validate the block ranges recorded in the aggsender database to enhance confidence
```
sqlite3 aggsender.sqlite
```

```
SELECT
    height,
    from_block,
    to_block
FROM
    certificate_info
WHERE
    status = 4
ORDER BY
    height DESC
LIMIT 1;
```

```
cast rpc --rpc-url "$l2_node_url" optimism_outputAtBlock $(printf "0x%x" $latest_settled_l2_block) | jq '.' > output.json
```

#### Retrieve the rollup configuration
```
cat << EOF > .env
L1_RPC="${l1_rpc_url}"
L1_BEACON_RPC="${l1_rpc_url}"
L2_NODE_RPC="${l2_node_url}"
L2_RPC="${l2_rpc_url}"
STARTING_BLOCK_NUMBER="${latest_settled_l2_block}"
EOF
```

```
docker run --rm -it \
  --platform linux/amd64 \
  -v "$(pwd)":/tmp/env \
  ghcr.io/agglayer/op-succinct/op-succinct:v2.3.1-agglayer \
  fetch-rollup-config \
  --env-file /tmp/env/.env \
  --output-dir /tmp/env
```

#### Initialize

```
cd agglayer-contracts
```

```
jq \
    --arg network_name ${network_name} \
    --arg admin_address ${admin_address} \
    --arg aggchain_vkey ${aggchain_vkey} \
    --arg aggchain_vkey_selector ${aggchain_vkey_selector} \
    --arg new_rollup_type_id ${new_rollup_type_id} \
    --slurpfile o ../output.json \
    --slurpfile s ../opsuccinctl2ooconfig.json \
    --slurpfile c ../combined.json \
   '.trustedSequencerURL = "http://" |
    .networkName = $network_name |
    .trustedSequencer = "@@replace" |
    .chainID = $c[0].l2ChainID |
    .rollupAdminAddress = "@@replace" |
    .gasTokenAddress = $c[0].gasTokenAddress |
    .deployerPvtKey = "@@replace" |
    .timelockDelay = "@@replace" |
    .rollupManagerAddress = $c[0].polygonRollupManagerAddress |
    .rollupTypeId = $new_rollup_type_id |
    .aggchainParams.aggchainManager = $admin_address |
    .aggchainParams.initOwnedAggchainVKey = $aggchain_vkey |
    .aggchainParams.initAggchainVKeySelector = $aggchain_vkey_selector |
    .aggchainParams.initParams.l2BlockTime = $s[0].l2BlockTime |
    .aggchainParams.initParams.startingOutputRoot = $o[0].outputRoot |
    .aggchainParams.initParams.startingBlockNumber = $o[0].blockRef.number |
    .aggchainParams.initParams.startingTimestamp = $o[0].blockRef.timestamp |
    .aggchainParams.initParams.submissionInterval = 1 |
    .aggchainParams.initParams.optimisticModeManager = $admin_address |
    .aggchainParams.initParams.aggregationVkey = $s[0].aggregationVkey  |
    .aggchainParams.initParams.rangeVkeyCommitment = $s[0].rangeVkeyCommitment |
    .aggchainParams.initParams.rollupConfigHash = $s[0].rollupConfigHash |
    .aggchainParams.vKeyManager = $admin_address |
    .realVerifier = true |
    .consensusContractName = "AggchainFEP"
'  ./tools/initializeRollup/initialize_rollup.json.example > ./tools/initializeRollup/initialize_rollup.json
```

> **trustedSequencer**: can be retrieved by invoking the trustedSequencer function of the rollup smart contract. See [important notes](./pp-to-fep.md#important-notes) for more details

> **timelockDelay**: the duration of the timelock. It's advisable to use a short delay for devnets (e.g., 60) for convenience

> **deployerPvtKey**: the private key used for executing smart contract operations


Review the content of `./tools/initializeRollup/initialize_rollup.json` and update the placeholders marked with `@@replace` with the appropriate values.

```
npm i
npx hardhat run ./tools/initializeRollup/initializeRollup.ts --network sepolia
```

## Deployment
### op-succinct-proposer
#### General info
| Item               | Description |
|--------------------|-------------|
| **Repository**      | [agglayer/op-succinct](https://github.com/agglayer/op-succinct) |
| **Docker Image**    | `ghcr.io/agglayer/op-succinct/op-succinct:v2.1.8-agglayer` |
| **Run Command**     | `/usr/local/bin/validity-proposer` |
| **Endpoints**       | This service exposes a gRPC endpoint (port `4446` in this example) |

#### Capacity
| Resource | Allocation  |
|----------|-------------|
| CPU      | 4000m       |
| Memory   | 16Gi        |
| Disk     | 500Gi       |

#### Dependencies
This service depends on a PostgreSQL database.

#### Environment variables
```
GRPC_ADDRESS=0.0.0.0:50001
AGGLAYER="true"
AGG_PROOF_MODE="compressed"
MAX_CONCURRENT_PROOF_REQUESTS="1"
MAX_CONCURRENT_WITNESS_GEN="1"
METRICS_ENABLED="true"
METRICS_PORT="7300"
OP_SUCCINCT_MOCK="false"
POLL_INTERVAL=20s
USE_CACHED_DB="false"
WITNESS_GEN_TIMEOUT="1200"
RUST_LOG="debug"
RANGE_PROOF_INTERVAL="1800"
PRIVATE_KEY="0xc797616a567ffd3f7d80f110f4c19900e55258ac2aa96d96ded790e0bd727458"
DATABASE_URL=@@replace
L1_BEACON_RPC=@@replace
L1_RPC=@@replace
L2_NODE_RPC=@@replace
L2_RPC=@@replace
L2OO_ADDRESS=@@replace
PROVER_ADDRESS=@@replace
NETWORK_RPC_URL=@@replace
NETWORK_PRIVATE_KEY=@@replace
```
> **DATABASE_URL**: PostgreSQL database connection URL

> **L1_BEACON_RPC**: l1 rpc url

> **L1_RPC**: l1 rpc url

> **L2_NODE_RPC**: op-node url

> **L2_RPC**: op-geth url

> **L2OO_ADDRESS**: rollup smart contract address

> **PROVER_ADDRESS**: must match the `trustedSequencer` and the address used by the `aggsender`. See [important notes](./pp-to-fep.md#important-notes) for further details

> **NETWORK_RPC_URL**: SP1 network url

> **NETWORK_PRIVATE_KEY**: SP1 network private key

> 💡 **PRIVATE_KEY** is an arbitrary private key. You may use a hardcoded value, as it is not currently used and is planned for deprecation in future releases

Update the placeholders marked with `@@replace` with the appropriate values.

### aggkit-prover
#### General info
| Item               | Description |
|--------------------|-------------|
| **Repository**      | [agglayer/provers](https://github.com/agglayer/provers) |
| **Docker Image**    | `ghcr.io/agglayer/aggkit-prover:0.1.0-rc.28` |
| **Run Command**     | `/usr/local/bin/aggkit-prover run --config-path /path/to/config.toml` |
| **Endpoints**       | This service exposes a gRPC endpoint (port `4446` in this example) |

#### Capacity
| Resource | Allocation  |
|----------|-------------|
| CPU      | 4000m       |
| Memory   | 16Gi        |
| Disk     | 500Gi       |

#### Config
Refer to the config example: [aggkit-prover.toml](config/aggkit-prover.toml)
#### Environment variables
```
NETWORK_RPC_URL=@@replace
NETWORK_PRIVATE_KEY=@@replace
```
> **NETWORK_RPC_URL**: SP1 network url

> **NETWORK_PRIVATE_KEY**: SP1 network private key

Update the placeholders marked with `@@replace` with the appropriate values.

### aggkit
#### General info
| Item               | Description |
|--------------------|-------------|
| **Repository**      | [agglayer/aggkit](https://github.com/agglayer/aggkit) |
| **Docker Image**    | `ghcr.io/agglayer/aggkit:0.3.0` |
| **Run Command**     | `/usr/local/bin/aggkit run --cfg=/path/to/config.toml --components=aggsender,aggoracle` |
| **Endpoints**       | This service doesn’t expose any endpoints |

#### Capacity
| Resource | Allocation |
|----------|------------|
| CPU      | 100m       |
| Memory   | 4Gi        |
| Disk     | 500Gi      |

#### Config
The config file looks almost identical; except for the AggSender section. In addition to the other fields, make it sure to add these new ones:
```
[AggSender]
...
Mode="AggchainProof"
AggchainProofURL="@@replace"
UseAgglayerTLS = true
RequireNoFEPBlockGap = true
...
```

> **AggchainProofURL**: aggkit-prover url

Update the placeholders marked with `@@replace` with the appropriate values.

### Important notes
- The `trustedSequencer` (set in the rollup smart contract) needs to match:
  - `PROVER_ADDRESS` environment variable of the op-succinct-proposer
  - Private key used by the aggsender in the `AggsenderPrivateKey` field
