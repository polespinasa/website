+++
title = "How the UTXO Set Can Help Us Synchronize a Bitcoin Node"
draft = false
date = 2025-02-13
tags = ["Nodes", "BitcoinCore", "Layer 1", "Bitcoin-cli"]
categories = ["Guides"]
about = "How to speed up the initial synchronization of the Bitcoin blockchain if you've previously run a node"
+++

We all have encountered at some point that our Bitcoin node stops working, the raspberry fails, the disk dies, or we simply need the disk for other purposes and we end up without a Bitcoin node. In the future when we want to set up a node again we find that we have to go through the IBD (*initial block download*) process to synchronize the blockchain. A heavy process, that can last in many cases days and that prevents us to use the node and to connect the wallets until the node is completely synchronized.

In this post we will see how we can advance to this situation and how we can take measures to be able, in the future, to synchronize our nodes quickly.

> :warning: **CLICK BAIT**
> *The node does not synchronize faster, but it does allow you to use it almost instantly using background synchronization!*

## First ideas

There are some ways to speed up the IBD process by sacrificing security by not verifying the blockchain. Although some are not recommended, they are explained in this first section.

### Assume valid

The `assumevalid` parameter in Bitcoin Core allows you to specify the node that up to a certain block height you do not want transactions to be verified and assumes that they are valid. Therefore, the process is simplified to just validating that the blocks are built correctly. Using this parameter during the initial synchronization greatly speeds up the process because the main factor that slows down the synchronization is the scripts verification.

However, with `assumevalid` we still need to download the whole chain to be able to use the node. Yes, it will go faster, but we still have to unload it until we reach the current status.

> :bulb: **Information!**
> *Bitcoin Core has `assumevalid` enabled by default at a pre-release block height.*

### Copy the blockchain

Another way to speed up the IBD process is to copy the files from one node to another. Specifically, you would copy the files from the `blocks` and `chainstate` directories.

This procedure may have some problems. To begin with, downloading the files from an unknown source is not recommended. If you have to carry out this procedure, do it from a node that you control. There may also be incompatibilities between versions of the node software if the way you create and deploy these files is different.

We discard this option because keeping redundant copies of the blockchain is not the optimal way to do it.

## How do we do it then? 🤔

Let's imagine that we could save a *checkpoint* representing the status of the chain at a certain block height. What information would this *checkpoint* have to contain? To verify transactions and spend bitcoins what we need is to know the unspent transactions in the present. Who keeps a record of these unspent transactions? The **UTXO set**!

The **UTXO set** is a database that represents the current status of unspent transaction outputs (UTXO). With the **UTXO set** we can verify that a transaction is valid without looking at the chain, since the entries of these have to spend *unspent transaction outputs* (UTXOs) and create new ones. In simpler words, it is a record that tells where the bitcoins are at this very moment.

### How do we use the UTXO set to synchronize the node?

If we have the current status of the chain, that is, the coins (UTXOs) that can be spent, we can make and verify transactions without problems. This works because **ALL** the bitcoins we have available to spend are represented in the **UTXO set**.

If we only need the **UTXO set** to make and verify transactions, can we simply load it at a very recent point, start using Bitcoin and, in the meantime, download the previous blocks?

Yes, the previous blocks can be downloaded in the background without affecting the wallets' ability to show the balance and spend funds.

> :bulb: **Information!**
> _This is the reason why the “pruned” nodes can also verify all the information, they only need the **UTXO set**!_

### Load a copy of the UTXO set

Let's suppose that we have a backup of the **UTXO set** in a file: `utxo.dat`. This backup of the **UTXO set** is made at block `840.000`, therefore, when we import it, our node will be set to `840.000` almost instantly.
To do this we can use the [command that introduces Bitcoin Core to version 26.0 “loadtxoutset”](https://bitcoincore.org/en/doc/28.0.0/rpc/blockchain/loadtxoutset/).

Let's start from the beginning. We start the Bitcoin node executing, as usual, the `bitcoind` binary. We will see that at the beginning it performs a presynchronization of the *headers* and then it synchronizes them. We have to wait for the synchronization of the *headers* to finish.

Presynchronization:
```
2025-02-18T10:53:47Z Pre-synchronizing blockheaders, height: 10000 (~1.18%)
2025-02-18T10:53:48Z Pre-synchronizing blockheaders, height: 12000 (~1.42%)
2025-02-18T10:53:49Z Pre-synchronizing blockheaders, height: 14000 (~1.66%)
2025-02-18T10:53:50Z Pre-synchronizing blockheaders, height: 16000 (~1.90%)
2025-02-18T10:53:50Z Pre-synchronizing blockheaders, height: 18000 (~2.14%)
2025-02-18T10:53:51Z Pre-synchronizing blockheaders, height: 20000 (~2.38%)
2025-02-18T10:53:52Z Pre-synchronizing blockheaders, height: 22000 (~2.63%)
2025-02-18T10:53:53Z Pre-synchronizing blockheaders, height: 24000 (~2.88%)
2025-02-18T10:53:53Z Pre-synchronizing blockheaders, height: 26000 (~3.13%)
2025-02-18T10:53:54Z Pre-synchronizing blockheaders, height: 28000 (~3.38%)
2025-02-18T10:53:55Z Pre-synchronizing blockheaders, height: 30000 (~3.62%)
2025-02-18T10:53:56Z Pre-synchronizing blockheaders, height: 32000 (~3.86%)
```

Synchronization:
```
2025-02-18T11:07:39Z Synchronizing blockheaders, height: 59379 (~7.13%)
2025-02-18T11:07:40Z Synchronizing blockheaders, height: 61379 (~7.37%)
2025-02-18T11:07:42Z Synchronizing blockheaders, height: 63379 (~7.60%)
2025-02-18T11:07:43Z Synchronizing blockheaders, height: 65379 (~7.84%)
2025-02-18T11:07:46Z Synchronizing blockheaders, height: 67379 (~8.07%)
2025-02-18T11:07:48Z Synchronizing blockheaders, height: 69379 (~8.30%)
2025-02-18T11:07:49Z Synchronizing blockheaders, height: 71379 (~8.53%)
2025-02-18T11:07:54Z Synchronizing blockheaders, height: 73379 (~8.76%)
2025-02-18T11:07:56Z Synchronizing blockheaders, height: 75379 (~9.00%)
2025-02-18T11:07:57Z Synchronizing blockheaders, height: 77379 (~9.23%)
2025-02-18T11:07:59Z Synchronizing blockheaders, height: 79379 (~9.47%)
2025-02-18T11:08:01Z Synchronizing blockheaders, height: 81379 (~9.70%)
```

Once the *headers* are synchronized, if we execute `bitcoin-cli getblockchaininfo` we will see that it starts to synchronize the blocks as usual:
```bash
$ bitcoin-cli getblockchaininfo
{
  "chain": "main",
  "blocks": 24503,
  "headers": 884325,
  ...
  "verificationprogress": 2.110628750476074e-05,
  "initialblockdownload": true,
  "chainwork": "00000000000000000000000000000000000000000000000000005fb85fb85fb8",
  "size_on_disk": 6748282,
  "pruned": false,
}
```

Once we have the node running and synchronizing we can import the **UTXO set** with the instruction `bitcoin-cli -rpcclienttimout=0 loadtxoutset utxo.dat`. Where `utxo.dat` is our backup of the **UTXO set**. ( I recommend to place the `utxo.dat` folder in the directory where you are saving the node data, by default it is `~.bitcoin`).

Before doing so it is recommended to use `bitcoin-cli setnetworkactive false`. This instruction improves performance, as it stops the networking. It must be re-enabled at the end of the download. (This is optional, if it is not done, it will work the same but with worse performance).

> :warning: **Warning**.
> *It is important to specify `-rpcclienttimout=0` to prevent the communication from being timed out, since the process can take a few minutes.*

It is common for the terminal to get stuck without acting like this:
![](/assumetxoutset/txoutset_timout.png#center)

While the command seems to remain stuck, you can see in the `bitcoind` logs how the node continues to synchronize normally and, eventually, you can see logs showing that it is loading the *coins* from the **UTXO set**.
![](/assumetxoutset/load_coins.png#center)

Once finished we will see that the RPC command confirms that we have imported the data:
```bash
bitcoin-cli -rpcclienttimeout=0 loadtxoutset utxo.dat 
{
  "coins_loaded": 176948713,
  "tip_hash": "0000000000000000000320283a032748cef8227873ff4872689bf23f1cda83a5",
  "base_height": 840000,
  "path": "/home/sliv3r/Escritorio/blockchain/utxo.dat"
}
```

If we look at the logs of `bitcoind`, we can see how it has updated the `chainstate` (the status of the chain) to the `840,000` block:
```
2025-02-18T14:55:18Z UpdateTip: new best=000000000000085b8326215f4334716986036cd7f22b474b4bb1d30e44abc7c6 height=190942 version=0x00000001 log2_work=68.448132 tx=5332360 date='2012-07-27T01:22:01Z' progress=0.004564 cache=264.9MiB(2106915txo)
2025-02-18T14:55:18Z [snapshot] validated snapshot (0 MB)
2025-02-18T14:55:18Z [snapshot] successfully activated snapshot 0000000000000000000320283a032748cef8227873ff4872689bf23f1cda83a5
2025-02-18T14:55:18Z [snapshot] (0 MB)
2025-02-18T14:55:18Z Opening LevelDB in /home/sliv3r/Escritorio/blockchain/chainstate
2025-02-18T14:55:18Z Opened LevelDB successfully
2025-02-18T14:55:18Z Using obfuscation key for /home/sliv3r/Escritorio/blockchain/chainstate: d06729b36c1332a2
2025-02-18T14:55:18Z [Chainstate [ibd] @ height 190942 (000000000000085b8326215f4334716986036cd7f22b474b4bb1d30e44abc7c6)] resized coinsdb cache to 0.4 MiB
2025-02-18T14:55:18Z [Chainstate [ibd] @ height 190942 (000000000000085b8326215f4334716986036cd7f22b474b4bb1d30e44abc7c6)] resized coinstip cache to 22.0 MiB
2025-02-18T14:55:18Z Cache size (277731616) exceeds total space (23068672)
2025-02-18T14:55:22Z Opening LevelDB in /home/sliv3r/Escritorio/blockchain/chainstate_snapshot
2025-02-18T14:55:22Z Opened LevelDB successfully
2025-02-18T14:55:22Z Using obfuscation key for /home/sliv3r/Escritorio/blockchain/chainstate_snapshot: ffaa87a9790b88bb
2025-02-18T14:55:22Z [Chainstate [snapshot] @ height 840000 (0000000000000000000320283a032748cef8227873ff4872689bf23f1cda83a5)] resized coinsdb cache to 7.6 MiB
2025-02-18T14:55:22Z [Chainstate [snapshot] @ height 840000 (0000000000000000000320283a032748cef8227873ff4872689bf23f1cda83a5)] resized coinstip cache to 418.0 MiB
```

And if we execute the command `bitcoin-cli getblockchaininfo` we will be able to see how effectively we have the node synchronized to the `840.000` block:

```bash
bitcoin-cli getblockchaininfo
{
  "chain": "main",
  "blocks": 840077,
  "headers": 884327,
  "bestblockhash": "000000000000000000018bd4948d21337c98ea27cd5586393a31c644ce9ab1ce",
  "bits": "17034219",
  "target": "0000000000000000000342190000000000000000000000000000000000000000",
  "difficulty": 86388558925171.02,
  "time": 1713621243,
  "mediantime": 1713616671,
  "verificationprogress": 0.8484856035034852,
  "initialblockdownload": true,
  "chainwork": "000000000000000000000000000000000000000075537cab1a1b8a216234e13a",
  "size_on_disk": 3395515205,
  "pruned": false,
  "warnings": [
    "This is a pre-release test build - use at your own risk - do not use for mining or merchant applications"
  ]
}

```

If you have disabled the p2p network, re-enable it with `bitcoin-cli setnetworkactive true`.
We look again at the `bitcoind` logs and see that now it is synchronizing the blocks between the `840,000` and the chain tip:
```
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000001caffc952e7de3b1c8b2f2d044e8e69955000ca6fb9e4 height=840436 version=0x20e00000 log2_work=94.879663 tx=993054314 date='2024-04-23T01:53:38Z' progress=0.849873 cache=458.5MiB(2943813txo)
2025-02-18T14:58:36Z UpdateTip: new best=000000000000000000014797e1aa0867732005d6cb11fad392e35fbc81f552bd height=840437 version=0x20000000 log2_work=94.879678 tx=993060398 date='2024-04-23T02:05:34Z' progress=0.849878 cache=459.1MiB(2947040txo)
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000002a0658cf7d5af3fadb3469df23e97de849a32991c0545 height=840438 version=0x3fffe000 log2_work=94.879693 tx=993066120 date='2024-04-23T02:13:23Z' progress=0.849883 cache=459.8MiB(2951620txo)
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000002349473de1b8f66f12b65cdeb870c211927865c856e41 height=840439 version=0x3fff0000 log2_work=94.879707 tx=993071846 date='2024-04-23T02:16:31Z' progress=0.849888 cache=460.5MiB(2956234txo)
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000000c6b9fd50913cf6d3d34536c06c1a029ba6c844cc8033 height=840440 version=0x20a00000 log2_work=94.879722 tx=993077310 date='2024-04-23T02:17:04Z' progress=0.849893 cache=461.2MiB(2961405txo)
2025-02-18T14:58:36Z UpdateTip: new best=000000000000000000033c2f66f551e44aea785f91a902b321c38703a158ed3a height=840441 version=0x20a00000 log2_work=94.879737 tx=993083167 date='2024-04-23T02:20:03Z' progress=0.849898 cache=461.9MiB(2965605txo)
2025-02-18T14
```

> :bulb: The current block in the chain, i.e. the last block in the chain with the most proof of work, is what we call the chain tip.

Once the node finishes synchronizing from block `840.000` to the current tip, in this case block `884.365`, we can see how it starts synchronizing the node from 0. It can be easily identified, since in the logs it is marked as *background validation*.

```
2025-02-18T20:18:41Z [background validation] UpdateTip: new best=000000000000001db5a1515a5f8534c941b1628f60466e6b709b3b320254afff height=208000 version=0x00000001 log2_work=69.023342 tx=8883319 date='2012-11-15T05:17:32Z' progress=0.007602 cache=58.2MiB(460376txo)
2025-02-18T20:19:04Z [background validation] UpdateTip: new best=000000000000048b95347e83192f69cf0366076336c639f9b7228e9ba171342e height=210000 version=0x00000002 log2_work=69.091539 tx=9344662 date='2012-11-28T15:24:38Z' progress=0.007996 cache=94.6MiB(728202txo)
2025-02-18T20:19:40Z [background validation] UpdateTip: new best=00000000000003d906e4131c39f7655b72df40146d2967f5d75113a09610de61 height=212000 version=0x00000002 log2_work=69.157547 tx=9819223 date='2012-12-13T01:33:36Z' progress=0.008403 cache=117.6MiB(927414txo)
2025-02-18T20:19:41Z Saw new header hash=000000000000000000019940b918fed6453c7afa80e03f1d837d1fdd8b31a8b9 height=884366
2025-02-18T20:19:45Z UpdateTip: new best=000000000000000000019940b918fed6453c7afa80e03f1d837d1fdd8b31a8b9 height=884366 version=0x2002a000 log2_work=95.452620 tx=1156361979 date='2025-02-18T20:19:40Z' progress=1.000000 cache=2.7MiB(19061txo)
```

Looking at the previous logs we can see that, although it is verifying very old blocks such as `210.000`, when a new block appears, `884.366` in this case, it immediately updates the tip of its chain and then continues with the verification of old blocks.

## Conclusions and cautions

With this process the node reaches a state where it is usable to consult the balance and create transactions much faster than synchronizing the whole chain. In a normal laptop, this whole process has taken me no more than 2 hours which is already a big improvement compared to the days that a normal synchronization can take.

> :warning: **Warning**.
> *It is very important not to download the __UTXO set__ from an unknown and untrusted source. The risk is very high, since a false representation of the __UTXO set__ can make you believe that you have been paid when it is not true. __You can be tricked and robbed__.*

Bitcoin Core tries to prevent this by restricting the import of the **UTXO set** to a block defined in the latest release (the `880,000` block as of this post) and verifying that the **UTXO set** is correct and has not been tampered. Skipping this check is not difficult and allows more experienced users to speed up this process even more. This post will not explain how to do it so as not to expose the readers to a possible attack.

> :warning: **Warning**.
> *When the 1st part of the post was written the block of the last release was the `840.000` and not the `880.000`, that's why the guide is about the `840.000`*.

## How to create or obtain a backup of the UTXO set?

The easiest way to get a backup of the **UTXO set** is with the magnet link: `magnet:?xt=urn:btih:559bd78170502971e15e97d7572e4c824f033492&dn=utxo-880000.dat&tr=udp%3A%2F%2Ftracker.bitcoin.sprovoost.nl%3A6969`.

As I said before, you should avoid downloading the **UTXO set** from unknown sources (**don't trust me!**), but well, there are different mechanisms to verify that the copy provided by this magnet link is correct:
- The link provided in the [PR Bitcoin Core](https://github.com/bitcoin/bitcoin/pull/31969) which introduces the parameters to the code.
- You can validate the hash of the copy with: `shasum -a 256 utxo-880000.dat` the result should give `43b3b1ad6e1005ffc0ff49514d0ffcc3e3e3ce671cc8d02da7fa7bac5405f89de4`.

### How do we generate our own backup?

To do this we need a node already synchronized to a block height higher than the backup. In this case higher than `880.000`.
> :warning: **Warning**.
> *The node cannot be pruned at a lower height than the height of the block on which the backup is made.*

Using the instruction `bitcoin-cli -rpcclienttimeout=0 dumptxoutset utxo.dat rollback`. By default this command will backup the **UTXO set** to the `~\.bitcoin` directory. This process may take a little while, because the node is getting desynchronized until it reaches the block of the last valid checkpoint.

[Here](https://x.com/sliv3r__/status/1891095297440280721) you can see a video of how a node is being desynchronized in this process.

Again you can validate that the backup is correct with the hash of the file:
```bash
shasum -a 256 utxo-880000.dat
43b3b1ad6e1005ffc0ff49514d0ffcc3e3ce671cc8d02da7fa7bac5405f89de4
```

Hey! Enjoy synchronizing nodes at the speed of light!!!
![](/assumetxoutset/lightspeed.gif#center)

**AND REMEMBER ---> DON'T TRUST, VERIFY. IF YOU DON'T RUN YOUR OWN NODE YOU ARE NOT USING BITCOIN WITHOUT A "TRUSTED" MIDLEMAN**