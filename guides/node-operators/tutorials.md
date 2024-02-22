# Tutorials

The specific steps to install and start es-node.

## Testnet Spec

- Layer 1: Sepolia testnet
- storage-contracts-v1: v0.1.0
- es-node: v0.1.6


## Options for running es-node

You can run es-node from a pre-built Docker image, a pre-built executable, or from the source code.

 - If you choose [the pre-built es-node executable](#from-pre-built-executables), you will need to manually install some dependencies such as Node.js and snarkjs.

 - If you have Docker version 24.0.5 or above installed, the quickest way to get started is by [using a pre-built Docker image](#from-a-docker-image).
 
 - If you prefer to build [from the source code](#from-source-code), you will also need to install Go besides Node.js and snarkjs.

## From pre-built executables

Before running es-node from the pre-built executables, ensure that you have installed [Node.js](#install-nodejs), [snarkjs](#install-snarkjs) and [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) if you are on Windows. 

Download the pre-built package suitable for your platform:

Linux x86-64 or WSL (Windows Subsystem for Linux):
```sh
curl -L https://github.com/ethstorage/es-node/releases/download/v0.1.6/es-node.v0.1.6.linux-amd64.tar.gz | tar -xz
```
MacOS x86-64:
```sh
curl -L https://github.com/ethstorage/es-node/releases/download/v0.1.6/es-node.v0.1.6.darwin-amd64.tar.gz | tar -xz
```
MacOS ARM64:
```sh
curl -L https://github.com/ethstorage/es-node/releases/download/v0.1.6/es-node.v0.1.6.darwin-arm64.tar.gz | tar -xz
```
Run es-node
```
cd es-node.v0.1.6
env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run.sh
```

## From a Docker image

Run an es-node container in one step:
```sh
docker run --name es  -d  \
          -v ./es-data:/es-node/es-data \
          -e ES_NODE_STORAGE_MINER=<miner> \
          -e ES_NODE_SIGNER_PRIVATE_KEY=<private_key> \
          -p 9545:9545 \
          -p 9222:9222 \
          -p 30305:30305/udp \
          --entrypoint /es-node/run.sh \
          ghcr.io/ethstorage/es-node:v0.1.6
```

You can check docker logs using the following command:
```sh
docker logs -f es 
```
## From source code

You will need to [install Go](#install-go) to build es-node from source code, and install [Node.js](#install-nodejs) and [snarkjs](#install-snarkjs) to run es-node.

Download source code and switch to the latest release branch:
```sh
git clone https://github.com/ethstorage/es-node.git
cd es-node
git checkout v0.1.6
```
Build es-node:
```sh
make
```

Start es-node
```sh
chmod +x run.sh && env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run.sh
```

With source code, you also have the option to build a Docker image by yourself and run an es-node container:

```sh
env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> docker-compose up 
```
If you want to run Docker container in the background and keep all the logs:
```sh
env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run-docker.sh
```


## Install dependencies

_Please note that not all steps in this section are required; they depend on your [choice](#options-for-running-es-node)._

### Install Go

Download a stable Go release, e.g., go1.21.4. 
```sh
curl -OL https://golang.org/dl/go1.21.4.linux-amd64.tar.gz
```
Extract and install
```sh
tar -C /usr/local -xf go1.21.4.linux-amd64.tar.gz
```
Update `$PATH`
```
echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.profile
source ~/.profile
```

### Install Node.js

Install Node Version Manager
```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
```
Close and reopen your terminal to start using nvm or run the following to use it now:
```sh
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```
Choose a Node.js version above 16 (e.g. v20.*) and install 
```sh
nvm install 20
```
Activate the Node.js version
```sh
nvm use 20
```
### Install snarkjs
```sh
npm install -g snarkjs
```

# Two phases after es-node launch

After the launch of ES node, it basically goes through two main stages.

## Data sync phase

The es-node will synchronize data from other peers in the network. You can check from the console the number of peers the node is connected to and, more importantly, the estimated syncing time.

During this phase, the CPUs are expected to be fully occupied for data processing. If not, please refer to [the FAQ](#how-to-tune-the-performance-of-syncing) for performance tuning on this area.

A typical log entry in this phase appears as follows:

```
INFO [01-18|09:13:32.564] Storage sync in progress progress=85.50% peerCount=2 syncTasksRemain=1@0 blobsSynced=1@128.00KiB blobsToSync=0 fillTasksRemain=30 emptyFilled=3,586,176@437.77GiB emptyToFill=608,127   timeUsed=1h48m7.556s  eta=18m20.127s

```
## Sampling phase 

Once data synchronization is complete, the es-node will enter the sampling phase, also known as mining.

A typical log entry of sampling during a slot looks like this:

```
INFO [01-19|05:02:23.210] Mining info retrieved                    shard=0 LastMineTime=1,705,634,628 Difficulty=4,718,592,000 proofsSubmitted=6
INFO [01-19|05:02:23.210] Mining tasks assigned                    shard=0 threads=64 block=10,397,712 nonces=1,048,576
INFO [01-19|05:02:26.050] The nonces are exhausted in this slot, waiting for the next block samplingTime=2.8s shard=0 block=10,397,712

```
When you see "The nonces are exhausted in this slot...", it indicates that your node has successfully completed all the sampling tasks within a slot. 
The "samplingTime" value informs you of the duration, in seconds, it took to complete the sampling process.

If the es-node doesn't have enough time to complete sampling within a slot, the log will display "Mining tasks timed out". For further actions, please refer to [the FAQ](#what-can-i-do-about-mining-tasks-timed-out).