# FAQ by Storage Providers

### What can I do about "Mining tasks timed out"? 

This is the most common issue storage providers encounter after finishing the data sync phase and entering the sampling phase.

Firstly, it's crucial to understand the importance of resolving this issue. The node attempts to perform 1,048,576 disk samplings every 12 seconds. For each sampling, it calculates a number based on the sample data to determine if it constitutes a valid proof (similar to Proof of Work, PoW). Thus, if the node cannot complete the 1,000,000 samplings in 12 seconds, you will encounter this warning log. As you can deduce, fewer samplings mean a lower chance of calculating a valid storage proof.

The underlying cause of this issue is the disk's inability to provide the necessary high read speed, which is why we recommend a NVMe SSD operating on PCIe 4.0. You should verify whether your disk meets this specification. Additionally, you can use 'fio' to test the reading speed:
```
fio --name=random-read --ioengine=libaio --direct=1 --rw=randread --bs=4k --size=128g --numjobs=1 --iodepth=1024
```
If the results indicate that the read IOPS are higher than 160k, then it should suffice.

If your disk meets the requirements but you still receive this warning, consider using the `--miner.threads-per-shard` flag. This allows specifying the number of threads for sampling each shard, potentially increasing the number of samplings achieved. For detailed instructions, check [here](#how-can-i-change-the-default-configurations) for the detailed operation.

### What should I do when the log frequently shows "i/o deadline reached" during data syncing?

This problem is often caused by a poor internet connection between your node and some peers, a common issue given that peers are located worldwide. Thus, it's likely you'll experience connection problems with some of them. The key detail to determine is the estimated syncing time shown in the log.
```
INFO [01-19|14:41:48.715] "Storage sync in progress" progress=14.46% peercount=21 tasksRemain=1@32 blobssynced=289,248@35.31GiB blobsTosync=1,711,107 timeused=2h42m13s etaTimeLeft=15h59m37s
```
If it's less than 24 hours, it should be acceptable. Additionally, it's possible that the data request size is too large compared to network performance, preventing all requested data from being returned in time.

You can change the initial size of request to a smaller value using `--p2p.request.size` for each peer, and the real request size would change according to network condition between a remote peer and local node. 
The current value in the `run.sh` is `4194304`. You can try adjusting it to `1048576`. Check [here](#how-can-i-change-the-default-configurations) for the detailed operation.

### How to tune the performance of syncing? 

To enhance syncing performance or limit the usage of CPU power, you can adjust the values of the following related flags with higher or lower values:

- The `--p2p.sync.concurrency` flag determines the number of threads that simultaneously retrieve shard data, with a default setting of 16.
- The `--p2p.fill-empty.concurrency` flag specifies the number of threads used to concurrently fill encoded empty blobs, with a default setting of the CPU number minus 2.

Check [here](#how-can-i-change-the-default-configurations) for the detailed operation.

### Why is the CPU fully utilized when syncing data? Does this imply that running an es-node requires high-end hardware? Additionally, is there a way to configure it to use less CPU power?

During data synchronization, the CPUs are fully utilized as the es-node is engaged in intensive data processing tasks like encoding and writing.

However, this high-intensity processing occurs primarily when large volumes of data need to be synchronized, such as during the initial startup of the es-node. Once the synchronization is complete, and the es-node enters the mining phase, the CPU usage decreases significantly. So, generaly speaking, the es-node does not require high-end hardware, particularly for the CPU.

If there is a need to conserve CPU power anyway, you can tune down the values of syncing performance related flags, namely `--p2p.sync.concurrency` and `--p2p.fill-empty.concurrency`. See [here](#how-to-tune-the-performance-of-syncing) for detailed information.

### How do I know whether I've got a mining reward?

You can observe the latest log that summarizes the mining status every minute, where succeeded indicates a successful mining submission.

```
"Mining stats since 2024-04-29 06:09:37" shard=0 succeeded=1 failed=0 dropped=0
```

The real-time logs that record the detailed information like reward and profit are like the following:
```
INFO [01-19|14:41:48.715] Mining transaction confirmed             txHash=62df87..7dbfbc
INFO [01-19|14:41:49.017] "Mining transaction success!      √"     miner=0x534632D6d7aD1fe5f832951c97FDe73E4eFD9a77
INFO [01-19|14:41:49.018] Mining transaction details               txHash=62df87..7dbfbc gasUsed=419,892 effectiveGasPrice=2,000,000,009
INFO [01-19|14:41:49.018] Mining transaction accounting (in ether) reward=0.01013071059 cost=0.0008397840038 profit=0.009290926583

```

You can also visit [the EthStorage dashboard](http://grafana.ethstorage.io) for real-time mining statistics.

Finally, pay attention to the balance change of your miner's address which reflects the mining income.

### Can I change the data file location?

Yes, you can. 

By default, when executed, the `run.sh` script will generate a data file named `shard-0.dat` to store shard 0 in the data directory specified by `--datadir`, located in the same folder as the `run.sh` script.

If necessary, you can choose an alternative location for data storage by specifying the full path of the file as the value of the `--storage.files` flag in the `run.sh` script.

Please refer to [configuration](/storage-provider-guide/configuration.md) for more details.

### What can I do about "The zkey file was not downloaded" error?

When you see the following message when running **run.sh**. you can manually download the [**blob_poseidon.zkey**](https://drive.google.com/file/d/1ZLfhYeCXMnbk6wUiBADRAn1mZ8MI_zg-/view) / [**blob_poseidon2.zkey**](https://drive.google.com/file/d/1G7LmOx7hNE5GHc-M6yOjVB3ZZ4J6xUYO/view) to `./build/bin/snarkjs/` folder and run it again. 
```
zk prover mode is 2
Start downloading ./build/bin/snarkjs/blob_poseidon2.zkey...
... ...
Error: The zkey file was not downloaded. Please try again.
```
If the error accurs while using Docker, you can mount the file from the host to the container using the `-v` parameter in `docker run`, just adjust the path accordingly:

```sh
docker run --name es  -d  \
    ...
    -v /host/path/to/blob_poseidon2.zkey:/es-node/build/bin/snarkjs/blob_poseidon2.zkey \
    ...

```

### How can I change the default configurations?

Just append the flag and value to the end of command that execute `run.sh`.

Take `--p2p.request.size` as example, the following command set its value to `1048576`:

```sh
env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run.sh --p2p.request.size 1048576
```
If you start es-node with docker, please note that the flag and value should be added afer the docker image name:

```sh
docker run --name es  -d  \
          -v ./es-data:/es-node/es-data \
          -e ES_NODE_STORAGE_MINER=<miner> \
          -e ES_NODE_SIGNER_PRIVATE_KEY=<private_key> \
          -p 9545:9545 \
          -p 9222:9222 \
          -p 30305:30305/udp \
          --entrypoint /es-node/run.sh \
          ghcr.io/ethstorage/es-node:v0.1.16 \
          --p2p.max.request.size 1048576
```

### Why does the "missing trie node" error occur?


```
Save blobs error err="missing trie node c4b8803bb72e5841894b5348db64a818f2e6481e51bb35bb49f8eb41c7ba57a8 (path ) state Bxc4be883bb72e5841094b5340db64a818f2e6481e51bb35bb49f8eb41c7ba57a8 is not available"
```
If you encounter the `missing trie node` error as described, it is likely that your es-node attempted to access the L1 state older than 256 blocks without success. This may be due to your execution layer endpoint not being an archive node.

If you are using a `BlockPI` endpoint, please navigate to the dashboard, access the detailed page of your API endpoint, and enable the `Archive Mode` within the `Advanced Features` section.


### Why should I use WSL to run Docker instead of Windows cmd? 

If you are using a Windows system, make sure to use WSL to run the docker command instead of running it directly from the command prompt (cmd). The reason for this is that in Windows cmd, it seems that the volume option (-v) for a relative path is not taking effect, which causes the data to be stored inside the Docker container. This can lead to data loss when the container needs to be upgraded. If an absolute path is provided, an "operation not supported" error occurs when creating the data file, which is caused by the program cannot allocate space for the file accross operating system.

### When running es-node in Docker on Windows, I want to store data files on a disk other than `C:`. How can I achieve this?

First of all, make sure you are running `docker run` command on WSL 2 (Ubuntu distro by default) instead of Windows cmd. 

Secondly, when start es-node with Docker image, we use the volumes option (-v) for storing data files outside containers to achive persistent storage. However the following attempt to mount an absolute path (e.g., `D:\es-data`) to the container would failed with "operation not supported" error.

```sh
# does not work!
docker run --name es  -d  \
          -v /mnt/d/es-data:/es-node/es-data \
          ...
```

The following approach will relocate the Ubuntu distro to the target volume (e.g. `D:`) so that `-v ./es-data:/es-node/es-data` can work properly.

By default the WSL system files are mapped to a specific directory on the `C` drive. To relocate the system to another volume (e.g., `D:`), execute the following command in `PowerShell`:

```sh
#  create new location in the target volume
> mkdir D:\wsl-ubuntu 
> wsl --shutdown
# verify Ubuntu is stopped
> wsl -l -v 
> wsl --export Ubuntu ubuntu.tar  
> wsl --unregister Ubuntu
> wsl --import Ubuntu D:\wsl-ubuntu\ .\ubuntu.tar --version 2
```
and reboot the computer at the end.


Now you can open a WSL window (Ubuntu on Windows) and start Docker container with the command [here](/storage-provider-guide/tutorials.md#from-a-docker-image).

### I am already running es-node. How can I update it to a newer version?

Firstly, you can [review the changes between releases](https://github.com/ethstorage/es-node/releases) to decide whether to upgrade or not. Then, depending on how you are currently running the es-node instance, different steps may need to be taken.

#### From pre-built executables

1. "Ctrl C" to stop the es-node process. 

2. Download new version (e.g., `es-node.v0.1.16`) of the pre-built package suitable for your platform using commands [here](/storage-provider-guide/tutorials.md#from-pre-built-executables).

3. Some files including data folder need to be moved from the directory of old build (e.g., `es-node.v0.1.15`) to the new one:

```sh
# replace v0.1.16 to your target version
cd es-node.v0.1.16

# replace v0.1.15 to the version you have
mv ../es-node.v0.1.15/es-data .
mv ../es-node.v0.1.15/esnode_* .
```
4. Launch es-node using the same command as the one previously used:

```sh
env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run.sh --l1.rpc <el_rpc> --l1.beacon <cl_rpc>
```
> ℹ️ **_Note:_** If you encounter an error indicating that the zkey file is not found, you may need to run the following command:

```sh
env ES_NODE_STORAGE_MINER=<miner> ./init.sh --l1.rpc <el_rpc>
```
The `init` command will download the necessary zkey file, and it will not damage or modify any existing data files.

Another option is to specify the file path of the zkey file using the `--miner.zkey` flag so that you don't need to run `init` and download zkey upon each upgrade:

```sh
env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run.sh --l1.rpc <el_rpc> --l1.beacon <cl_rpc> \
  --miner.zkey <absolute path to the zkey>
```

#### From a Docker image

1. As the data folder is located on the host where you execute the `docker run` command, you can safely stop and remove the old version of the container:

```sh
docker stop es
docker remove es
```

2. Then start a new container based on the new version of the es-node Docker image with the following command, just make sure the `<version>` in `ghcr.io/ethstorage/es-node:<version>` is correct:

```sh
docker run --name es -d \
          -v ./es-data:/es-node/es-data \
          -v ./zkey:/es-node/build/bin/snark_lib/zkey \
          -e ES_NODE_STORAGE_MINER=<miner> \
          -e ES_NODE_SIGNER_PRIVATE_KEY=<private_key> \
          -p 9545:9545 \
          -p 9222:9222 \
          -p 30305:30305/udp \
          --entrypoint /es-node/run.sh \
          ghcr.io/ethstorage/es-node:v0.1.16 \
          --l1.rpc <el_rpc> \
          --l1.beacon <cl_rpc>
```

#### From source code

1. "Ctrl C" to stop the es-node process. 

2. Switch to the correct branch of the source code (e.g., you want to move to `v0.1.16`):

```sh
cd es-node

# fetch new branches
git fetch

# replace 'v0.1.16' to your target version
git checkout v0.1.16
```
3. build and launch es-node

```sh
make

env ES_NODE_STORAGE_MINER=<miner> ES_NODE_SIGNER_PRIVATE_KEY=<private_key> ./run.sh --l1.rpc <el_rpc> --l1.beacon <cl_rpc>
```
