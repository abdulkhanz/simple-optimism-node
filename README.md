# Simple Optimism Node

I think it's really important that people start running their own Optimism nodes.
I've created this repository to make that process as simple as possible.
You should be relatively familiar with running commands on your machine.
Let's do it!

**Note:** The Optimism Goerli testnet was upgraded to Bedrock on Thursday January 12th 2023.
These directions apply to the Bedrock version.
[See here](https://github.com/smartcontracts/simple-optimism-node) for directions for mainnet which is still pre-Bedrock at writing.

Please also report any bugs that you find, it'll help speed up the process of getting to production Goerli Bedrock nodes.
Thank you!

## Required Software

- [docker](https://docs.docker.com/engine/install/)

## Recommended Hardware

- 16GB+ RAM
- 500GB+ disk (HDD works for now, SSD is better)
- 10mb/s+ download

## Approximate Disk Usage

Usage as of 2022-09-21:

- Archive node: ~800gb
- Full node: ~60gb


## Installation and Setup Instructions

Instructions here should work for MacOS and most Linux distributions.

### Clone the Repository

```sh
git clone https://github.com/smartcontracts/simple-optimism-node.git
cd simple-optimism-node
```

### Torrents

`simple-optimism-node` uses torrents to download certain configuration files and databases as a decentralized alternative to a central server hosting everything.
By default, the torrent port is 6881 but can be configured with the `PORT__TORRENT` environment variable.

If you have a firewall, you will need to expose the torrent port to the internet to be able to download things:

```sh
uwf allow 6881
```

### Optional steps

<details>
    
<summary>

#### Configure docker as a non-root user

</summary>

If you're planning to run Docker as a root user, you can safely skip this step.
However, if you're using Docker as a non-root user, you'll need to add yourself to the `docker` user group:

```sh
sudo usermod -a -G docker `whoami`
```

You'll need to log out and log in again for this change to take effect.

</details>    


<details>
    
<summary>
    
#### Setting a docker data directory

</summary>    
    
This step might be useful for anyone who was confused as I was about how to make Docker point at disk other than your primary disk.
If you'd like your Docker data to live on a disk other than your primary disk, create a file `/etc/docker/daemon.json` with the following contents:

```json
{
    "data-root": "/mnt/<disk>/docker_data"
}
```

Make sure to restart docker after you do this or the changes won't apply:

```sh
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Confirm that the changes were properly applied:

```sh
docker info | grep -i "Docker Root Dir"
```

</details>    


### Current limitations

We are in the middle of the Bedrock upgrade. 
While the basic functionality is available when you create a node using this document, some advanced features are not yet supported:

- Legacy database migration is not yet supported, all Goerli Bedrock nodes will be started from a pre-migrated database downloaded via torrent.
- Nodes seed the pre-migrated database torrent by default. Upload limits cannot yet be configured.
- No functional metrics dashboard.
- No legacy node request forwarding.
- No fault detector or healthcheck service.

## Configure the node

Make a copy of `.env.example` named `.env`.

```sh
cp .env.example .env
```

Open `.env` with your editor of choice and fill out the environment variables listed inside that file.
Only the following variables are required for Bedrock:

| Variable Name                           | Description                                                               |
|-----------------------------------------|---------------------------------------------------------------------------|
| `NETWORK_NAME`                          | Network to run the node on (currently only `goerli` is supported)         |
| `NODE_TYPE`                             | Type of node to run (`full` or `archive`)                                 |
| `BEDROCK_SOURCE`                        | Where to get the Bedrock database from (only `download` supported for now)   |
| `OP_NODE__RPC_ENDPOINT`                 | L1 node that the op-node will get chain data from
| `OP_NODE__RPC_TYPE`                     | Type of RPC that you're connected to, available options are: `alchemy`, `quicknode`, `infura`, `parity`, `nethermind`, `debug_geth`, `erigon`, and `basic` |

You can also modify any of the optional environment variables if you'd wish, but the defaults should work perfectly well for most people.

### The OP_NODE__RPC_TYPE variable

The `OP_NODE__RPC_TYPE` environment variable tells the `op-node` service what type of RPC it's connected to.
This allows `op-node` to make more efficient requests with custom RPCs that may only be available on some RPCs.
It is HIGHLY recommended to set this variable to the appropriate provider or you may experience rate limits and/or excessive requests to your RPC.

## Initial synchronization

You start the node with this command:
    
```sh
docker compose up -d
```
    
When you start it it will synchronize (get the transaction history).    

To know when the synchronization is over use this command to get the latest synchronized block:
    
```sh
docker compose logs op-geth | grep "Chain head was updated" | tail -1
```
    
Part of this log entry is the age on the latest block.
When this value is sufficiently small (a few minutes), the replica is ready for use.    
    
## Operating the node

### Start

```sh
docker compose up -d
```

Will start the node in a detatched shell (`-d`), meaning the node will continue to run in the background.
    
        
### Stop

```sh
docker compose down
```

Will shut down the node without wiping any volumes.
You can safely run this command and then restart the node again.

### Wipe

```sh
docker compose down -v
```

Will completely wipe the node by removing the volumes that were created for each container.
This is a complete reset, and synchronization will be required after it is done.


### Logs

```sh
docker compose logs <service name>
```

Will display the logs for a given service.
You can also follow along with the logs for a service in real time by adding the flag `-f`.

The available services are:
- `op-geth`
- `op-node`
- `influxdb`
- `torrent`

<!--
- `prometheus`
- `grafana`
-->

### Software updates

```sh
docker compose pull
```

Will download the latest images for any services where you haven't hard-coded a service version.
Updates are regularly pushed to improve the stability of Optimism nodes or to introduce new quality-of-life features like better logging and better metrics.
I recommend that you run this command every once in a while (once a week should be more than enough).
If you intend to maintain an Optimism node for a long time, it's also worth subscribing to the [Optimism Public Changelog](https://changelog.optimism.io/) via either [RSS](https://changelog.optimism.io/feed.xml) or the [optimism-announce@optimism.io mailing list](https://groups.google.com/a/optimism.io/g/optimism-announce).
    

## Using the node
    
To use the node specify the URL `http://<node IP>:9991` as the JSON RPC URL.
    
<!-- 
    
## Additional components
    
### Metrics dashboard

Grafana is exposed at [http://localhost:3000](http://localhost:3000) and comes with one pre-loaded dashboard ("Simple Node Dashboard").
Simple Node Dashboard includes basic node information and will tell you if your node ever falls out of sync with the reference L2 node or if a state root fault is detected.

Use the following login details to access the dashboard:

* Username: `admin`
* Password: `optimism`

Navigate over to `Dashboards > Manage > Simple Node Dashboard` to see the dashboard, see the following gif if you need help:

![metrics dashboard gif](https://user-images.githubusercontent.com/14298799/171476634-0cb84efd-adbf-4732-9c1d-d737915e1fa7.gif)


### Healthcheck

When you run your Optimism node using these instructions, you will also be running two services that monitor the health of your node and the health of the network.
The Healthcheck service will constantly compare the state computed by your node to the state of some other reference node.
This is a great way to confirm that your node is syncing correctly.

### Fault Detector

The Fault Detector service will continuously scan the transaction results published by the Optimism Sequencer and cross-check them against the transaction results that your node generated locally.
**If there's ever a discrepancy between these two values, please complain very loudly!**
This either means that the Sequencer has published an invalid transaction result or there's a bug in your node software and an Optimism developer needs to know about it.
In the future, this service will trigger Cannon, the fault proving mechanism that Optimism is building as part of its Bedrock upgrade.

The Fault Detector exposes several metrics that can be used to determine whether your node has detected a discrepancy including the `is_currently_diverged` gauge. The Fault Detector also exposes a simple API at `localhost:$PORT__FAULT_DETECTOR_METRICS/api/status` which returns `{ ok: boolean }`. You can use this API to monitor the status of the Fault Detector from another application.

-->
