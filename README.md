# Block Producer Configuration

Background information: In February 2020, an architecture was developed to help EOS Mainnet scale and avoid having failing transactions clog up the network and have empty blocks. The high level overview is here: https://eosnation.io/eos-mainnet-update-new-node-architecture-greatly-improves-eos-reliability/

As of November 2023, we have removed the block relay nodes as nodeos leap 4.0.0 an up now prioritizes block propogation over transactions.

As of February 2024, this is the current configuration of nodeos to support this architecture. Nodes need to be running nodeos leap **v5.0.0** or later.

The Producer API and Chain API must not be exposed to public. Use a reverse proxy to expose the /v1/chain/... APIs, but keep the others private.

## block relay (blocks peer node)

- have 2 of these nodes for redundancy
- connect to other block producers or other trusted entities
- connect to BP nodes

```
wasm-runtime = eos-vm-jit
eos-vm-oc-enable = true
chain-state-db-size-mb = 131072
http-max-response-time-ms = 300
read-mode = head
database-map-mode = mapped_private
p2p-accept-transactions = false
http-validate-host = false
p2p-max-nodes-per-host = 2
agent-name = INSERT NAME OF BP HERE
max-clients = 0
net-threads = 5
verbose-http-errors = true
abi-serializer-max-time-ms = 2000
block-log-retain-blocks = 172800

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
```

## transaction sentry (transaction barrier node)

- have 2 of these nodes for redundancy
- connect to other block producers or other trusted entities
- connect to BP nodes

```
wasm-runtime = eos-vm-jit
chain-state-db-size-mb = 131072
http-max-response-time-ms = 300
disable-subjective-api-billing = false
disable-subjective-p2p-billing = false
subjective-account-decay-time-minutes = 60
read-mode = head
database-map-mode = mapped_private
http-validate-host = false
p2p-max-nodes-per-host = 2
agent-name = INSERT NAME OF BP HERE
max-clients = 0
net-threads = 5
verbose-http-errors = true
abi-serializer-max-time-ms = 2000
block-log-retain-blocks = 172800

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
```

## block producer

- have 2 of these nodes for redundancy, one primary and one backup
- connect **only** to _block relay_ and _transaction sentry_ defined above (4 connections total). **do not connect to any other nodes**

```
wasm-runtime = eos-vm-jit
chain-state-db-size-mb = 131072
disable-subjective-api-billing = false
disable-subjective-p2p-billing = false
read-mode = speculative
subjective-account-decay-time-minutes = 60
database-map-mode = mapped_private
http-max-response-time-ms = 300
http-validate-host = false
p2p-max-nodes-per-host = 2
agent-name = INSERT NAME OF BP HERE
max-clients = 0
net-threads = 4
read-only-threads = 0
verbose-http-errors = true
abi-serializer-max-time-ms = 2000
block-log-retain-blocks = 0

subjective-cpu-leeway-us = 36000
producer-name = INSERT ACCOUNT OF PRODUCER
signature-provider = INSERT PRODUCER KEY
actor-blacklist = INSERT MULTIPLE LINES HERE

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
plugin = eosio::producer_plugin
```

## RAM configuration

We have used 

```
chain-state-db-size-mb = 131072
database-map-mode = mapped_private
```

This loads all the blockchain data into RAM and makes computing transactions fast. However, this requires at least 32GB physical RAM and 96GB of SWAP space (SWAP space required if you don't have more than 128GB RAM).

On some cloud providers (like AWS), they limit the disk I/O. If you load all the blockchain state in memory, then you avoid any problems with this  I/O limiting. You should work to increase resources on your server in the future, as the blockchain is expected to grow with the renewed interest that ENF is generating.

You can see the amount of RAM and SWAP on your server as follows:

```
$ free -m
               total        used        free      shared  buff/cache   available
Mem:           31651       23338        1487           6        6824        7865
Swap:         143050       10493      132556
```

If you have used a larger value of `chain-state-db-size-mb` previously, then nodeos will not shrink the size (it will print a warning the log file about it and your node will fail to start because of not enough RAM). Start from a snapshot to shrink the size. https://snapshots.eosnation.io/eos-v6/latest


Note: state can also be put into tempfs to achieve similar perfomance improvements, that is not discussed here.

## Extra node: API Node

```
wasm-runtime = eos-vm-jit
chain-state-db-size-mb = 131072
http-max-response-time-ms = 300
read-mode = head
database-map-mode = mapped_private
p2p-accept-transactions = false
disable-api-persisted-trx = true
http-validate-host = false
p2p-max-nodes-per-host = 2
agent-name = INSERT NAME OF BP HERE
max-clients = 0
net-threads = 5
http-threads = 8
verbose-http-errors = true
abi-serializer-max-time-ms = 2000
http-server-address = 0.0.0.0:8888
enable-account-queries = true

plugin = eosio::http_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_api_plugin
plugin = eosio::producer_api_plugin
plugin = eosio::db_size_api_plugin
```

You will notice that OC is not enabled, and `p2p-accept-transactions = false`.  This avoids processing lots of transactions and the RPC requests for transactions are billed at the same value that will be used on the BP node (assuming hardware CPU is the same). `enable-account-queries = true` can be set to enable lookup of account information. This is important on public API nodes. 

As a reminder, the Producer API and Chain API must not be exposed to public. Use a reverse proxy to expose the /v1/chain/... APIs, but keep the others private.

Connect the API node to both the "block relay" and "transaction sentry" nodes.

![Nodes-Graphic](https://user-images.githubusercontent.com/36178664/187374756-67edb1b5-7836-4056-99fe-53c8138a6649.png)
