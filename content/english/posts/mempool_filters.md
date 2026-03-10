+++
title = 'A technical overview and opinion about mempool filters'
date = 2025-09-02
draft = false
tags = ["Nodes", "Mempool", "P2P", "Mining", "Policy"]
categories = ["Opinion"]
+++

### What is the Mempool?

The simplest definition for the mempool is: ***the mempool*** *is a cache that stores unconfirmed consensus valid transactions.*

#### **But why do we need a mempool?**

The mempool is an important resource for each node and it mainly allows the peer-to-peer transaction relay network.

Every \~10 minutes when a new block is found, nodes without a mempool experience a bandwidth spike and a computational-intensive period validating each transaction in the block. On the other hand, it is likely that nodes that implement a mempool have already seen all or most of the transactions inside a new block and have them stored in their mempools.

With [Compact block relay](https://bitcoinops.org/en/topics/compact-block-relay/) ([BIP-152](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki)) these nodes just download the header of the block plus some hints of which transactions are inside the block, then the node reconstructs the block picking the transactions from the mempool and asks for the ones missing from the node that sent the block header. This is much more efficient and faster than requesting the whole block as less data needs to be sent, and the transaction validation has been already done before. Using a mempool and compact blocks, dramatically increases the network-wide block propagation speed and thus reduces the risk of stale blocks.

The mempool is also useful to estimate the fees you need to pay in order to get a transaction mined in the time range you want. The block space is sold in a fee-based auction market, keeping a mempool helps to have an idea of what others are willing to pay for block space and what bids have been successful in the past.

For more information about why do we have a mempool read [Waiting for confirmation #1: why do we have a mempool?](https://bitcoinops.org/en/newsletters/2023/05/17/#waiting-for-confirmation-1-why-do-we-have-a-mempool)

### Consensus Rules vs Policy Rules

When we talk about filters, we mean the rules that decide which transactions are **standard**. If a transaction is standard, it makes it into our mempool and gets relayed across the network. We don't add transactions to our mempool neither we relay transactions that we consider non-standard. 

Note that in the previous definition we don't talk about blocks or the blockchain, this is because **those policies are not applied to transactions in blocks**. If these policies were to be applied to transactions in blocks we would have multiple chain splits due to discrepancies between nodes.

> ***A simple example:*** Suppose that mempool policy rules are also applied to blocks. Alice and Bob having a different mempool policy, Alice only accept transactions that move at least 1 BTC and Bob does not have any minimum amount. At some point, when a miner finds a block with some transaction with less than 1 BTC Alice would discard that block while Bob would accept it. At the end they would be in two different versions of the blockchain!

**The rules that are applied to transactions in blocks are called consensus rules** and this rules must be enforced in a compatible way by all nodes. If that is not the case then the blockchain is likely to get [forked](https://bitcoin.stackexchange.com/questions/30817/what-is-a-soft-fork-what-is-a-hard-fork-what-are-their-differences). Some example of this rules could be that a transaction cannot have outputs that spend more money than what it has in the inputs, transactions cannot double spend coins, among others.

#### **So what is the role of the policy rules (filters)?**

Your node is computer connected to a peer-to-peer network with a lot of other unknown entities that may launch a denial of service (DoS) attack crafting messages that cause the node to run out of memory and crash, or spend its computational resources and bandwidth on meaningless data instead of accepting new blocks. Because of these entities being unknown by design, your node cannot know whether he is connecting to a honest or malicious peer and cannot effectively ban them to avoid being attacked. Because of this, it is imperative to implement policies against DoS to limit the cost of running a full node.

> ***A real simple example:*** An attacker could create a transaction with a lot of signatures of which all but the last signature are valid. This would make your node spend many resources and time validating the transaction until it fails just at the end. This is prevented with a policy rule that enforces a limit on the number of signature operations that can be in a transaction. Note that because this rule only applies to transactions you receive outside blocks, if it ends in a valid block your node will validate and store that transaction.

For a further explanation about what are the policy rules and why are they used see [Waiting for confirmation #5: Policy for Protection of Node Resources](https://bitcoinops.org/en/newsletters/2023/06/14/#waiting-for-confirmation-5-policy-for-protection-of-node-resources).

### OP\_RETURN

OP\_RETURN is an opcode in Bitcoin's scripting system that allows for the inclusion of some arbitrary data in a transaction. This opcode immediately marks the transaction output as unspendable, ensuring that the output can't be used as an input in a subsequent transaction.

The maximum weight of the data you can embed using OP\_RETURN is mainly limited by the block size (a consensus rule). Historically there has been a policy rule that limited the amount of data that could be added. The latest rule limited OP\_RETURN outputs to 83 bytes. This limit is raised on Bitcoin Core, by default, to 100000 bytes as there's a policy rule for the maximum transaction weight. Remember that a policy rule only affects the transactions you will relay, it doesn't make a transaction invalid if seen in a block.

#### **Does allowing bigger OP\_RETURN incentivize spam?**

I don't see why it would. It is for sure probable that we will see now more OP\_RETURNS with bigger amount of data. Also, it is doubtful that OP\_RETURN limit removal increase the overall amount of "data payloads" as it is more expensive to use an OP\_RETURN than to use other protocol like [inscriptions](https://docs.ordinals.com/inscriptions.html). As Vojtěch points in the [Bitcoin stack exchange](https://bitcoin.stackexchange.com/a/122322/145161), OP\_RETURN stops being economically worth if the data to be inscribed surpasses the 143 bytes.

#### **Will an increased OP\_RETURN limit cause UTXO set growth?**

For context, the UTXO set is a database that nodes have where they store the unspent transaction outputs, that is to say, all the available coins to be spent. The size of the UTXO set matters for performance and decentralization reasons, the larger the UTXO set is, the larger the requirements to run a node are.

Because OP\_RETURN makes an ouput unspendable it doesn't need to be stored in the UTXO set, thus not making it bigger. In other words is the cleanest and healthiest way to embed data into the blockchain.

#### **Why is better than other spam methods**

Note that when we say better we mean healthiest way. The first reason was already mentioned in the previous paragraph; creating OP\_RETURN outputs does not bloat the UTXO set, but there are other good points.

Another important point is that it shows the data directly in the output. Some other methods embed the data using SegWit, or Taproot scripts, those scripts are "secret" until the output is spent, once they are spent the data is revealed in the next transaction's inputs. This means that in order to embed data they need to create two transactions instead of just one. This way of work may provide a reason to create dummy outputs.

One extra point to consider is that OP\_RETURN scripts do not have the witness discount.

### Filters

As mentioned previously the main role of filters (policy rules) is to prevent nodes from being DoS attacked, but not to prevent transactions to get into blocks.

#### **How can filters be skipped?**

As long as there is an economic incentive to do so, skipping those policy rules (filters) is easy and can be done in multiple ways:

- Submit directly to mining pool's API: Mining pools sometimes have the option to submit directly transactions to them without relaying into the peer-to-peer network. This allows the sender to avoid all filters and get into a block with certain security.
- Use permissive node with preferential peering: Nodes with less strict policy rules (like [Libre Relay](https://geyser.fund/project/librerelay)) will forward those filtered transactions and, eventually, they can end up in miners mempools anyway. Sometimes even they implement preferential peering to ensure there's a route between those nodes and miners. This way transactions reach miners even if a large percentage of nodes in the network filters them.

#### **Do filters work?**

One of the main problems in the discussion taken is the disagreement on the definition of "work". On one side we have seen arguments saying that filters do work because they difficult the propagation of transactions, pushing them to go through less convenient paths like the ones mentioned in the previous section. On the other side we have seen arguments saying that filters do not work because they can not avoid transactions getting into blocks.

In my opinion filters do work, but they work for the purpose they are made for, **prevent nodes from being DoS attacked.** They propose is not to avoid transactions to get into the block. So if what we want is to not have data embedded into the blockchain we should go and find a solution that address that specific problem.

#### **Why filters are a bad solution and may be harmful?**

Let's recap a bit about what we have already talked in order to answer this question with all the necessary context. We have a mempool in our nodes, which is a cache of unconfirmed transactions that we think that may end up in the next blocks. Because we need blocks to be propagated fast, in order to avoid stale blocks, we have a protocol called Compact Blocks. This protocol allows us to reconstruct and broadcast the new blocks faster because we already have the block transactions in our mempool.
Because Bitcoin is a permission-less network we cannot know if our peers are malicious until they are connected to us which means we need to have protection measures to avoid being DoS attacked. We protect our self filtering on what we will broadcast to the rest of the network. Those filters do not avoid transactions to get into blocks and only makes the propagation of transactions more difficult, but not impossible.


**So why are filters a harmful solution?** **Because** **they push towards mining centralization!** There are two reasons why filters push towards mining centralization, but before going into them let's define mining centralization.

In this case when we talk about mining being centralized we are talking about who create the block templates and not who hashes them. Here we can identify two different actors, the block template creator, in other words; the one who decide which transactions are getting into blocks, and the hashers, all those users with ASICs, bitaxes, etc. mining on top of some pool that does the template creation job.

There are two different reasons why filters push towards mining centralization.

- If your node filters transactions, the compact block protocol will fail to reconstruct the block fast as some information will be missing. This will lead to slower block propagation and will increase the possibility of stale blocks. In this scenario the smaller miners are the ones most negatively impacted because big miners are likely to be better connected between them. This makes that probably most part of the network hashrate will mine on top of the bigger miner block, thus throwing away all the work done by the small miner.
- If a user chooses to submit a transaction out of band, instead of relaying on the peer-to-peer network, using for example a public API from a miner pool, it will probably choose a big pool. This is because submitting it to a big pool increases the chances of the transaction being mined faster. This transactions out of band are normally accompanied by an extra fee. This extra fee is what we would consider [MEVil](https://bluematt.bitcoin.ninja/2024/04/16/stop-calling-it-mev/). Summarizing, MEVil causes mining centralization because the blocks from those big pools, where users submit out of band transactions, earn more money in fees than the other ones. This pushes hashers towards pointing their hashrate to those pools to maximize profits.

An example of filters not working can be seen with the recent (second half 2025) sub 1sat/vb transactions drama. TLDR -- A big bunch of the miners [started accepting transactions that paid less than 1 sat/vB](https://x.com/mononautical/status/1947530080475091159) while nodes policy rules filtered those transactions. The outcome? Those transactions were mined anyway (filters didn't work) and compact block was broken, block propagation [was slowed down significantly](https://github.com/bitcoin/bitcoin/pull/33106#issuecomment-3155627414).

In summary not only filters cannot avoid "spam" getting into the blockchain but also they are harmful for the decentralization of mining. They are not a solution for this problem, they are shooting ourselves in the foot.

### Did Segwit and Taproot cause this?

No, for sure no. OP\_RETURN was there before SegWit and Taproot and it could be used to embed data in the blockchain. And the other methods that are being used could use P2SH.

Although SegWit and Taproot aren't cause, I would not discard them contributing indirectly to the growth of "spam". SegWit came with a discount that made it cheaper to embed data and Taproot raised the amount of data that could be attached to one input, as it does not have the 10,000 bytes limit that P2WSH have.

### Are Datum and Sv2 the solution?

Datum and Sv2 are both good improvements towards decentralizing mining by letting users create their own block templates and still split the reward of the blocks found. The problem is that they only resolve the **"how do we decentralize block template creation?"** question. The "how we do it" doesn't matter if we have strong centralizing forces towards centralized mining pools.

Hashers (people with mining devices such as ASICs, BitAxes, etc.) are usually incentivized by economics, at the end of the day mining is a business and the objective is to extract the maximum value with as little cost as possible while maximizing the profits. If there is a big pool mining blocks with extra reward because of out of band transactions(MEVil), hashers have a strong incentive to point their mining power to that pool. This makes Datum and Sv2 useless and irrelevant.

Another problem with Datum, Sv2 and filters is the game theory behind collaborating with unknowns. Ideally a hasher will want to pool with other miners that also work to maximize profits. Why would a miner that maximize profits collaborate with someone who excludes some transactions thereby not getting all possible rewards? In order to collaborate people has to be driven by the same objectives and incentives. Trusting that hashers will do the same work for less money while others collect that money just for ideals is not realistic and leaves the outcome of the centralizing mining problem in the hands of the unknown.

In conclusion, I truly believe that Datum and Sv2 are great solutions and important improvements, but their final goal is decentralize mining to avoid big pools censoring transactions. They are not a solution versus mining centralization that is driven by economic incentives.