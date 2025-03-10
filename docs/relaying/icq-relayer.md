# ICQ Relayer

## Overview

[Interchain Queries](/neutron/interchain-queries/overview) allow smart contracts to make queries to a remote chain. An ICQ Relayer is a required component for making them possible. It acts as a facilitator between the Neutron chain and a querying chain, gathering queries that are needed to be performed from the Neutron, actually performing them, and eventually making the results available for the Neutron's smart contracts. These three main responsibilities are described in details below.

If you are a smart contracts developer and up to develop your dApp on Neutron, you will most likely need your own ICQ Relayer to manage your Interchain Queries. 

### Queries gathering

All registered Interchain Queries and their parameters are stored in the eponymous module and available by its [query interface](/neutron/interchain-queries/client#queries). The Relayer utilises the module's interface in order to initialise the performing list of queries. This is how the Relayer maintains the list of queries to be executed:

- on initialisation, the ICQ module `RegisteredQueries` query is executed with the `RELAYER_REGISTRY_ADDRESSES` parameter used for the `Owners` field;
- during the rest of the run, the Relayer listens to the ICQ module's `query_update` and `query_removed` [events](/neutron/interchain-queries/events) and modifies the queries list and parameters correspondingly.

The Relayer also listens to the Neutron's `NewBlockHeader` events that are used as a trigger for queries execution. Since each query has its own `update_period`, the Relayer tracks queries execution height and executes only the queries which update time has come.

### Queries execution

When the update time comes for a query, the Relayer runs the specified query on the remote chain:
* in case of a KV-query, the Relayer just [reads](https://github.com/neutron-org/cosmos-query-relayer/blob/4542045ab24d2735890e70d4dc525677d5f30c8a/internal/proof/proof_impl/get_storage_values.go#L11)
necessary KV-keys from the remote chain's storage with [Merkle Proofs](https://github.com/cosmos/cosmos-sdk/blob/ae77f0080a724b159233bd9b289b2e91c0de21b5/docs/interfaces/lite/specification.md). Neutron will need these proofs to [verify](https://github.com/neutron-org/neutron/blob/49c33ff43122cb12ee20e98493e0e2439a94f928/x/interchainqueries/keeper/msg_server.go#L217) validity of KV-results on results submission;
* in case of a TX-query, the Relayer makes a query to the target chain's [Tendermint RPC](https://docs.tendermint.com/v0.33/app-dev/indexing-transactions.html#querying-transactions) 
to search transactions by message types, events and attributes which were emitted during transactions execution and were 
[indexed](https://docs.tendermint.com/v0.33/app-dev/indexing-transactions.html) by Tendermint. More about Tx query parameters syntax [in the dedicated section](/neutron/interchain-queries/messages#register-interchain-query). When Relayer submits transactions search results to Neutron chain, it **DOES NOT** include events into result (even if events were used for the query), because [events are not deterministic](https://github.com/tendermint/tendermint/blob/bff63aec83a4cfbb3bba253cfa04737fb21dacb4/types/results.go#L47), therefore they can break blockchain consensus. One more important thing about TX queries is that the Relayer is made the way it only searches for and submits transactions within the trusting period of the Tendermint Light Client. Trusting period is usually calculated as `2/3 * unbonding_period`. Read more about Tendermint Light Client and trusted periods [at this post](https://blog.cosmos.network/light-clients-in-tendermint-consensus-1237cfbda104).

### Results submission

Relayer submits a query result as the following depending on the Relayer's configuration:
- simply sending it to the Neutron's Interchain Queries module which handles it by storing the result in the blockchain state (KV queries with `RELAYER_ALLOW_KV_CALLBACKS`=false);
- sending it to the Neutron's Interchain Queries module which handles it by storing the result in the blockchain state and passing the result to the owner smart contract (KV queries with `RELAYER_ALLOW_KV_CALLBACKS`=true);
- passing it to the smart contract that has registered the query (TX queries).

This means that it's the Relayer who pays gas for these actions. Note that KV queries submission are straightforward and therefore cheap whereas TX ones and KV callbacks also include smart contract call and their cost may vary significantly.

#### A bit of technical details about TX submission

The KV queries are submitted in a fire-and-forget way, i.e. they are submitted once per `update_period` span and never retried forcibly (e.g. on a submission error). The TX queries are a bit more tricky: since they are not stored in the Neutron chain and simply passed to smart contracts, it's needed that each tx is passed and handled by the smart contract only once. The Relayer uses the `BroadcastTxSync` messages broadcast type to maintain balance between performance and submission detalisation, but this means that the submission result is not waited for. To achieve both submission speed and only one submission handling, the Relayer fires submission messages, remembers the query result as sent, and then in the background retrieves the submission result for the query. If it turns to be a success, the TX is saved as fully processed and will not be sent to the smart contract again. Otherwise, this tx will be marked as failed and will not be sent to the smart contract again during this run. Instead, to prevent repeated submission of transactions which can't be successfully handled by the smart contract, the retry will only be possible on Relayer restart.

## Configuration

This section contains description for all the possible config values that the Relayer supports. For example values see the [.env.example](https://github.com/neutron-org/neutron-query-relayer/blob/main/.env.example) file in the Relayer's repository.

### Neutron chain node settings

- `RELAYER_NEUTRON_CHAIN_RPC_ADDR` — RPC address of a Neutron node to interact with (e.g. get events and to submit results);
- `RELAYER_NEUTRON_CHAIN_REST_ADDR` — REST address of a Neutron node to interact with (e.g. get registered queries list);
- `RELAYER_NEUTRON_CHAIN_CHAIN_ID` — Neutron chain ID;
- `RELAYER_NEUTRON_CHAIN_HOME_DIR` — path to keys directory;
- `RELAYER_NEUTRON_CHAIN_SIGN_KEY_NAME` — name of the key pair to be used by the Relayer;
- `RELAYER_NEUTRON_CHAIN_TIMEOUT` — timeout for Neutron RPC calls;
- `RELAYER_NEUTRON_CHAIN_GAS_PRICES` — the price for a unit of gas used by the Relayer;
- `RELAYER_NEUTRON_CHAIN_GAS_LIMIT` — the maximum price to be paid for a single submission;
- `RELAYER_NEUTRON_CHAIN_GAS_ADJUSTMENT` — gas multiplier used in order to avoid underestimating;
- `RELAYER_NEUTRON_CHAIN_CONNECTION_ID` — Neutron chain connection ID;
- `RELAYER_NEUTRON_CHAIN_CLIENT_ID` — IBC client ID for an IBC connection between the Neutron chain and the target chain;
- `RELAYER_NEUTRON_CHAIN_DEBUG` — flag to run neutron chain provider in debug mode;
- `RELAYER_NEUTRON_CHAIN_KEYRING_BACKEND` — described [here](https://docs.cosmos.network/master/run-node/keyring.html#the-kwallet-backend);
- `RELAYER_NEUTRON_CHAIN_OUTPUT_FORMAT` — Neutron chain provider output format;
- `RELAYER_NEUTRON_CHAIN_SIGN_MODE_STR` — described [here](https://docs.cosmos.network/master/core/transactions.html#signing-transactions), also consider use short variation, e.g. `direct`.

### Target chain node settings

- `RELAYER_TARGET_CHAIN_RPC_ADDR` — RPC address of a target chain node to interact with (e.g. send queries);
- `RELAYER_TARGET_CHAIN_CHAIN_ID` — target chain ID;
- `RELAYER_TARGET_CHAIN_ACCOUNT_PREFIX` — target chain account prefix;
- `RELAYER_TARGET_CHAIN_VALIDATOR_ACCOUNT_PREFIX` — target chain validator account prefix;
- `RELAYER_TARGET_CHAIN_TIMEOUT` — timeout for target chain RPC calls;
- `RELAYER_TARGET_CHAIN_CONNECTION_ID` — target chain connection ID;
- `RELAYER_TARGET_CHAIN_CLIENT_ID` — IBC client ID for an IBC connection between the Neutron chain and the target chain;
- `RELAYER_TARGET_CHAIN_DEBUG` — flag to run neutron chain provider in debug mode;
- `RELAYER_TARGET_CHAIN_OUTPUT_FORMAT` — target chain provider output format.

### Relayer application settings

- `RELAYER_REGISTRY_ADDRESSES` — a list of comma-separated smart-contract addresses (registered query owners) for which the Relayer processes interchain queries. If empty, literally all registered queries are processed which is usable if you are up to deploy a public Relayer;
- `RELAYER_ALLOW_TX_QUERIES` — if true, Relayer will process tx queries (if `false`, Relayer will ignore them). A true value here is mostly usable for a private Relayer because TX queries submission is quite expensive;
- `RELAYER_ALLOW_KV_CALLBACKS` — if `true`, will pass proofs as sudo callbacks to contracts. A true value here is mostly usable for a private Relayer because KV query callbacks execution is quite expensive. If false, results will simply be submitted to Neutron and become available for smart contracts retrieval;
- `RELAYER_MIN_KV_UPDATE_PERIOD` — minimal period of queries execution and submission. This value is usable for a public Relayer as a rate limiter because it roughly overrides the queries `update_period` and force queries execution not more often than `N` blocks;
- `RELAYER_STORAGE_PATH` — path to leveldb storage, will be created on the given path if it doesn't exist. It is required if `RELAYER_ALLOW_TX_QUERIES` is `true`;
- `RELAYER_CHECK_SUBMITTED_TX_STATUS_DELAY` — delay in seconds between TX query submission and the result handling checking (more about this in the [TX submission section](#a-bit-of-technical-details-about-tx-submission));
- `RELAYER_QUERIES_TASK_QUEUE_CAPACITY` — capacity of the channel that is used to send messages from subscriber to Relayer. Better set to a higher value to avoid problems with Tendermint websocket subscriptions;
- `RELAYER_PROMETHEUS_PORT` — the port on which Prometheus metrics API is available.
- `RELAYER_INITIAL_TX_SEARCH_OFFSET` - Only for transaction queries. If set to non zero and no prior search height exists, it will initially set search height to (last_height - X). One example of usage of it will be if you have lots of old tx's on first start you don't need. Keep in mind that it will affect each newly created transaction query.

### Logger configuration

As it is said in the Relayer's [readme](https://github.com/neutron-org/neutron-query-relayer#logging), the Relayer uses a little bit modified version of Uber's [zap.Logger](https://github.com/uber-go/zap). This modification allows logger configuration via env parameters. See the [logger configuration guide](https://github.com/neutron-org/neutron-logger#configuration-via-env-variables) readme for more information.

## Prerequisites

Before running the Relayer application for production purposes, you need to create a wallet for the Relayer, top it up, and set up the configuration (refer to the [Configuration](#configuration) section). Also you will most likely need to deploy your own RPC nodes of Neutron and the chain of interest.

- [How to deploy your own Neutron RPC node](/neutron/build);
- [How to how to prepare a target chain RPC node for Relayer's usage](/relaying/target-chain).

### Setting up Relayer wallet

1. The keyring folder for Relayer's usage is configured by the `RELAYER_NEUTRON_CHAIN_HOME_DIR` variable. The easiest way is to run `neutrond keys` from the cloned [neutron repository](https://github.com/neutron-org/neutron) and get the default value from the `--keyring-dir` flag:

```
neutrond keys
Keyring management commands. These keys may be in any format supported by the
Tendermint crypto library and can be used by light-clients, full nodes, or any other application
that needs to sign with a private key.
...
Flags:
      --keyring-dir string       The client Keyring directory; if omitted, the default 'home' directory will be used
...
Global Flags:
      --home string         directory for config and data (default "/Users/your-user/.neutrond")
```

2. Then execute `neutrond keys add relayer --keyring-backend test` to create an account in the default keyring directory;
3. Use `relayer` as the `RELAYER_NEUTRON_CHAIN_SIGN_KEY_NAME`, `test` as the `RELAYER_NEUTRON_CHAIN_KEYRING_BACKEND`, and pass the keyring directory as a volume to the Relayer's docker container using the keyring path in the container as the `RELAYER_NEUTRON_CHAIN_HOME_DIR`;
4. Get the Relayer's wallet address and top its balance up. If you're running the Relayer on the testnet, use the official Neutron faucet. For the mainnet, get some NTRN for the address.

## Running the Relayer

1. Make sure you've finished the [Configuration](#configuration) part;
2. Build Relayer's docker image from the Relayer's folder:

```
make build-docker
```

3. Run Relayer in a docker container way:

```
docker run --env-file .env.example -p 9999:9999 neutron-org/neutron-query-relayer
```

Notes:
- `-p 9999:9999` exposes the port that allows access to the Relayer's metrics powered using Prometheus. The container's port will be the same as the `RELAYER_PROMETHEUS_PORT` value that is `9999` by default. Use another value if you are up to use a different port;
- add keyring passing to the volumes list. For example, assign `RELAYER_NEUTRON_CHAIN_HOME_DIR=/keyring` and run the app as:

```
docker run --env-file .env.example -v /Users/your-user/.neutrond:/keyring -p 9999:9999 neutron-org/neutron-query-relayer
```
