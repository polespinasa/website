+++
title = 'Grouphug - Batching transactions without interaction between users'
date = 2024-05-21
draft = false
tags = ["Sighash", "UTXO management", "PSBT", "Coinjoin", "Layer 1"]
categories = ["Theory"]
+++

With the intention of trying to help make Bitcoin transactions more economical and inspired by the implementation of [Peach](https://peachbitcoin.com/) made by [Czino](https://x.com/capoczino) GroupHug has been born. A _transaction batching_ system that allows to share between users the fees of a transaction without the need of coordination between users.

This post explains what GroupHug is, how it works technically and what advantages and disadvantages it has.


### The Bitcoin Grouphug

GroupHug is a _transaction batching_ solution that allows to create joint transactions between users similar to a coinjoin without interaction or coordination between users, but with some limitations.

The main benefit of this model is the reduction of fees. The savings can be up to 10% approximately. At first it may not seem like a very high saving, but in an environment with very high fees it can be quite significant.

In order to calculate the savings that GroupHug can give us, what we have to do is to calculate the weight in _virtual bytes_ that each field of a transaction has, the savings will be the relation between the number of users and the weight, within a transaction, of the common fields, that is to say, everything that is not the inputs and the outputs.

We know that a P2WPKH-secured transaction with 1 input and 1 output weighs 109.25 virtual bytes.
If we stay only with the common fields we have:

| Field | Size (Bytes) | Size (WU)
| ------------ | ------------ | ------------ |
| Tx version        | 4 | 16 |
| Inputs Count      | 1 | 4 |
| Outputs Count     | 1 | 4 |
| nLockTime         | 4 | 16 |
| Segwit Marker     | 1 | 1 |
| Segwit Flag       | 1 | 1 |

We have that the common fields weigh 12 bytes. We pass them to _Weight Units_ by multiplying by 4 the fields that are not segwit (Tx Version, Input Count, Output Count and nLockTime) and we get that the size in WU is 42 WU. To pass from WU to _virtual bytes_ we divide by 4 and get that the common fields occupy 10.5 vb.

We know that the common fields represent 10.5/109.25 = ~9.6%.
Therefore, with GroupHug we can spread the cost of about 10% of the transaction among the users.

> :bulb: For more information on how to calculate the transaction size, please refer to the following links: [BitcoinStackExchange](https://bitcoin.stackexchange.com/questions/92689/how-is-the-size-of-a-bitcoin-transaction-calculated) and [BitcoinOptech](https://bitcoinops.org/en/tools/calc-size/).

### Low-level operation

The main functionality that GroupHug uses is what is known as SigHash. Although it is not a term that is well known to regular Bitcoin users, the fact is that it is used in every input and every signature of a transaction.

The SigHash is basically a _flag_ that is used to indicate what is being signed. The idea that in order to spend an UTXO a valid signature must be provided is well known, but what exactly is signing? The SigHash comes to indicate that. There are different ones, but the most common is the `ALL` SigHash where in each signature ALL the inputs and ALL the outputs are signed.

> bulb: For more information about the different SigHash types, please refer to the following links: [MasteringBitcoin](https://github.com/bitcoinbook/bitcoinbook/blob/6c472dd00b649b18b6ca6bbcc8ba23775619ce08/ch06.asciidoc#signature-hash-types-sighash). The images used below are also from the Mastering Bitcoin book.

The SigHash used by GroupHug is: `SINGLE | ANYONECANPAY`. This is made up of two parts. `SINGLE` means that the signature applies to all inputs but only to one output, specifically to the same level output.
![](/grouphug/sighash_guide.png#center)
![](/grouphug/single.png#center)

As it is seen in the previous figure, with this we manage to add outputs without invalidating the previous signatures, but not inputs, since these are all signed and, therefore, the transaction is rigid.

In order to be able to add inputs the `ANYONECANPAY` is added. As shown in the figure with the combination of these we indicate that the signatures go 1 to 1 inputs with outputs and, as a result, it is possible to add inputs and outputs indiscriminately without invalidating the signatures already made.

![](/grouphug/single_anyonecanpay.png#center)

This is therefore the basis of non-interaction between GroupHug users. As the signatures are 1 to 1 between inputs and outputs it is possible to group transactions from different users **already signed** without invalidating their signatures.

There is one more detail to take into account and that is that the signatures besides signing inputs and outputs sign some common fields of the transaction such as the `Tx version` and the `LockTime`.

### Advantages and disadvantages

The advantages are easy to identify, save fees when making a transaction by sharing the cost of the bytes of the common fields of a transaction between different users.

It should be noted that it would be a mistake to understand GroupHug as a coinjoin (Whirlpool or Wasasbi style) or a mixer. Even if inputs and outputs from different users are joined, they are not shuffled and match 1 to 1 in the same order. Moreover, they are linked by the signature. Each input from a user is signing the output from that same user. Therefore, the GroupHug **IS NOT A PRIVACY TOOL**.

As a disadvantage we have that the transactions that users enter in GroupHug must have the same number of inputs and outputs. In addition, the transactions must meet certain requirements such as specifying the same `Tx version` and the same `LockTime` in order to be able to group them without invalidating the signatures. And they must be signed with the sighash `SINGLE | ANYONECANPAY`, all these requirements are a UX problem.

These UX problems could be solved if the wallet detects that the transaction contains the same number of inputs and outputs, gives the user the option to send it to a GroupHug and in doing so the wallet itself completes the transaction complying with all the above mentioned requirements.

> :warning: **DEV ALERT!
> *If any wallet developer reads this, already knows... 😉 You can write to me at the email address on this page, in order to implement it, or you can find me on twitter*.

### Community

You can check the code of my GroupHug implementation on the [GitHub](https://github.com/polespinasa/bitcoin-grouphug) repository.
You can also find it running on the [Barcelona Bitcoin Only](https://x.com/bcnbitcoinonly) community server:
- Mainnet: https://grouphug.bitcoinbarcelona.xyz/
- Testnet3: https://grouphug.testnet.bitcoinbarcelona.xyz/
- Signet: https://grouphug.signet.bitcoinbarcelona.xyz/ (this is the [BBO signet](https://x.com/oomahq/status/1785685345536806986) not the 'official' one)














high five \\._./ 