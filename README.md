# Setup Sequencer Node With docker-compose

**Recommend using Kubernetes instead**

# Prerequisite

**You must have enough AWS and Linux operation experience to make sure your sequencer reliable and stable.**

## Submit your Sequencer information

Generate sequencer key

```sh
docker run --rm -it -v ./tmp:/var/lib/themis metisdao/themis init
```

You will get the key in following files

```
tmp
├── config
│   ├── app.toml
│   ├── config.toml
│   ├── genesis.json
│   ├── node_key.json
│   ├── priv_validator_key.json
│   └── themis-config.toml
└── data
    └── priv_validator_state.json

3 directories, 7 files
```

you need to backup `node_key.json` and `priv_validator_key.json` and ensure their security

Run following commands to get the public key and address of your sequencer

```console
$ docker run --rm -it -v ./tmp:/var/lib/themis metisdao/themis show-account
{
    "address": "0xBF578472333179247657a6023D7418D6Dc792Ef3",
    "pub_key": "0x042e1a3735f93ddbad4f697dc576e37b159f31f344808bff2a18269b228fc83663cfb66da6462fd258b5ef09d4a532a41ef96d0d589f1e0401317c21534c0239ad"
}
$ docker run --rm -it -v ./tmp:/var/lib/themis metisdao/themis show-privatekey
{
    "priv_key": "0x...REDACTED..."
}
```

NOTE:

the `priv_key` is the same with `SEQ_PRIV` env in the l2geth service.

the `address` is the same with `SEQ_ADDRESS` env in the l2geth service.

Fork [sequencer info](https://github.com/MetisProtocol/metis-sequencer-resources) and refer to its README to add your sequencer info

Make a pull request and wait for it to merge

## Setup l2geth and l1dtl service

1. Spin up an ec2 instance

System Requirements

- c5.2xlarge, the root volume has 40Gi space at least
- ebs gp3, 3000 iops, 200 MB/s throughput(**IMPORTANT**), 500Gi size, mount the volume to `/data` path
- request an EIP and associate it to the instance
- add following security group rules

| Port  | Description           | Protocol | Source    |
| ----- | --------------------- | -------- | --------- |
| 30303 | L2geth P2P Port (UDP) | UDP      | 0.0.0.0/0 |
| 30303 | L2geth P2P Port (TCP) | TCP      | 0.0.0.0/0 |

2. Install docker

https://docs.docker.com/engine/install/

3. Login to the instance

4. Copy the l2geth config

copy `<NETWORK_NAME>/l2geth/*` directory and `docker-compose-l2geth.yaml` file to `/opt/metis/l2geth/`

5. Add addition `.env` file

Add following environment values to `.env` file

5.1 your Ethereum rpc endpoint, and the rpc node must have entie transaction and event logs

for metis-sepolia, it's the sepolia rpc endpoint

for metis-andromeda, it's the mainnet rpc endpoint

```
DATA_TRANSPORT_LAYER__L1_RPC_ENDPOINT=
```

5.2 your sequencer private key

refer to the note of `Submit your Sequencer information` section to get them

```
SEQ_ADDRESS=
SEQ_PRIV=
```

5.3 copy the `.env` to `/opt/metis/l2geth/`

6. Start

Run following command in the `/opt/metis/l2geth/` on the instance

```sh
mv docker-compose-l2geth.yaml docker-compose.yaml
docker compose up -d
```

Note: the healthcheck is not the same between the testnet and the mainnet, check out the healthcheck filed in the docker-compose file for the details.

7. Add `docker-autoheal` service

Run following command on the instance

```sh
docker run -d --name autoheal --restart=always \
    -e AUTOHEAL_CONTAINER_LABEL=all \
    -v /var/run/docker.sock:/var/run/docker.sock \
    willfarrell/autoheal
```

It can restart the l2geth services if they're not healthy.

8. Check for the l2geth to be synced

```
curl http://localhost:8545 -X POST -H "Content-Type: application/json" --data '{"method":"rollup_getInfo","params":[],"id":1,"jsonrpc":"2.0"}' | jq
{
   "jsonrpc":"2.0",
   "id":1,
   "result":{
      "mode":"sequencer",
      "syncing":false,
      "ethContext":{
         "blockNumber":19686775,
         "timestamp":1713497023
      },
      "rollupContext":{
         "index":16675761,
         "queueIndex":37232,
         "verifiedIndex":16675761
      }
   }
}
```

if the `.result.syncing` is false, your l2geth is syncd

## Setup themis node

1. Spin up **another new** ec2 instance

System Requirements

- c5.2xlarge, the root volume has 40Gi space at least
- ebs gp3, 3000 iops, 100Gi size, mount the volume to `/data` path
- request an EIP and associate it to the instance
- add following security group rules

| Port  | Description          | Protocol | Source    |
| ----- | -------------------- | -------- | --------- |
| 26656 | Themis Node P2P Port | TCP      | 0.0.0.0/0 |

2. Prepare themis config for your node

2.1 Replace the genesis and themis-config file

```sh
cp <NETWORK_NAME>/themis/genesis.json tmp/config
cp <NETWORK_NAME>/themis/themis-config.toml tmp/config
```

2.2 Edit `tmp/config/config.toml` and update `persistent_peers` field.

Please note, the peer info is not public in this repo for now, you can only get it after you're whitelisted.

3. Install docker

https://docs.docker.com/engine/install/

4. Login to the instance

5. Copy the themis config

5.1 update the `<NETWORK_NAME>/themis/bridge.env`

```
METIS_RPC_URL=http://<the-l2geth-instance-private-ip>:8545
ETH_RPC_URL=<YOUR_ETH_RPC_ENDPOINT>
```

5.2 create `<NETWORK_NAME>/themis/node.env`

```
METIS_RPC_URL=http://<the-l2geth-instance-private-ip>:8545
ETH_RPC_URL=<YOUR_ETH_RPC_ENDPOINT>
```

5.3 Copy the local `tmp/*` to `/data/themis/` on the instance

5.4 Copy compose files

copy `docker-compose-themis.yaml` and `<NETWORK_NAME>/themis/*.env` to `/opt/metis/themis` on the instance

6. Start

Run following command in the `/opt/metis/themis` on the instance

```sh
mv docker-compose-themis.yaml docker-compose.yaml
docker compose up -d
```

7. Check for themis to be synced

You will see many such logs in the console if your themis is not synced.

```
$ docker compose logs -f --tail=10 bridge
Waiting for themis to be synced
```

Run following command to check if the node is synced

```console
$ curl http://localhost:26657 -X POST -H "Content-Type: application/json" --data '{"method":"status","params":{},"id":1,"jsonrpc":"2.0"}' | jq '.result.sync_info'
{
  "latest_block_hash": "3D5C64A9D141FADBBA4F122DE3EF3B5882E5B9F945B73CEDDB4F234A03111CE2",
  "latest_app_hash": "64BC4D400A68ECEAB237E8C1825C90F8C2884598830C0C4542CB00F4024F7936",
  "latest_block_height": "3719739",
  "latest_block_time": "2024-04-26T12:29:24.792569284Z",
  "catching_up": false
}
```

if the `catching_up` is true or `latest_block_time` is less than 10s away from the current time, your node is syncd

## Setup sequencer exporter

1. Spin up an ec2 instance

- c5.large, the root volume has 20Gi space at least
- request an EIP and associate it to the instance
- add following security group rules

| Port | Description          | Protocol | Source    |
| ---- | -------------------- | -------- | --------- |
| 9090 | The sequencer export | TCP      | 0.0.0.0/0 |

2. Install docker

https://docs.docker.com/engine/install/

3. Create `config.yaml`

```yaml
sequencer:
  your_sequencer_name:
    l1dtl: "http://the-l1dtl-private-ip:7878"
    l2geth: "http://the-l2geth-private-ip:8545"
    themis: "http://the-themis-private-ip:1317"
```

4. Login to the instance

5. Copy config

copy `config.yaml` and `docker-compose-exporter.yaml` file to `/opt/metis/exporter/`

6. Start

Run following command in the `/opt/metis/exporter` on the instance

```sh
mv docker-compose-exporter.yaml docker-compose.yaml
docker compose up -d
```

7. Send the ip to Metis Devops team

you can use you own prometheus to scrape the metrics as well, refer to [rules.yaml](./rules.yaml) for the alert rule example.

## What's next

1. waiting for the l2geth and the themis are both synced
2. Add prometheus and alertmanager to monitor your sequencer instance
3. Lock Metis on https://sequencer.metis.io
