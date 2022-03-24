# Block Producer Configuration

Background information: In February 2020, an architecture was developed to help EOS Mainnet scale and avoid having failing transactions glog up the network and have empty blocks. The high level overview is here: https://eosnation.io/eos-mainnet-update-new-node-architecture-greatly-improves-eos-reliability/

As of March 2022, this is the current configuration of nodeos to support this architecutre. Nodes need to be running nodeos **v2.0.12** (or later 2.0.*) or **v2.1.0**

## block relay (blocks peer node)

- have 2 of these nodes for redundancy
- connect to other block producers or other trusted entities

```
wasm-runtime = eos-vm-jit
eos-vm-oc-enable = true
chain-state-db-size-mb = 32768
reversible-blocks-db-size-mb = 2048
http-max-response-time-ms = 300
read-mode = head
database-map-mode = heap
p2p-accept-transactions = false
http-validate-host = false
p2p-max-nodes-per-host = 10000
agent-name = INSERT NAME OF BP HERE
allowed-connection = any
max-clients = 0
net-threads = 5
verbose-http-errors = true
abi-serializer-max-time-ms = 2000

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
```

## transaction sentry (transaction barrier node)

- have 2 of these nodes for redundancy
- connect to other block producers or other trusted entities

```
wasm-runtime = eos-vm-jit
chain-state-db-size-mb = 32768
reversible-blocks-db-size-mb = 2048
http-max-response-time-ms = 300
disable-subjective-billing = false
read-mode = head
database-map-mode = heap
http-validate-host = false
p2p-max-nodes-per-host = 10000
agent-name = INSERT NAME OF BP HERE
allowed-connection = any
max-clients = 0
net-threads = 5
sync-fetch-span = 2500
verbose-http-errors = true
abi-serializer-max-time-ms = 2000

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
```

## block producer

- have 2 of these nodes for redundancy, one primary and one backup
- connect only to block relay and transaction relay defined above

```
wasm-runtime = eos-vm-jit
chain-state-db-size-mb = 32768
reversible-blocks-db-size-mb = 2048
disable-subjective-billing = false
http-max-response-time-ms = 300
http-validate-host = false
p2p-max-nodes-per-host = 2
agent-name = INSERT NAME OF BP HERE
allowed-connection = specified
max-clients = 0
net-threads = 2
sync-fetch-span = 2500
verbose-http-errors = true
abi-serializer-max-time-ms = 2000
cpu-effort-percent = 40
last-block-cpu-effort-percent = 20
last-block-time-offset-us = -400000
max-transaction-time = 35
producer-name = INSERT NAME OF PRODUCER
signature-provider = INSERT PRODUCER KEY
actor-blacklist = INSERT MULTIPLE LINES HERE

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
plugin = eosio::producer_plugin
```

## additional configuration

If running nodeos 2.1, the old blocks can be auto-removed by adding the following:

```
blocks-log-stride = 10000
max-retained-block-files = 50
blocks-archive-dir = ""
```
