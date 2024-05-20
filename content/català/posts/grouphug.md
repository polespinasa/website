+++
title = 'Grouphug - Agrupació de transaccions sense interacció entre usuaris'
date = 2024-04-16
draft = true
tags = ["Sighash", "UTXO management", "PSBT", "Coinjoin", "Layer 1"]
categories = ["Teoria"]
+++

Amb l'intenció d'intentar ajudar a fer transaccions de Bitcoin més econòmiques i inspirat en l'implementació de [Peach](https://peachbitcoin.com/) feta per en [Czino](https://x.com/capoczino) neix el GroupHug. Un sistema de _transaction batching_ que permet compartir entre usuaris les tarifes d'una transacció sense necesitat de coordinació entre usuaris.

En aquest post s'explica què és el GroupHug, com funciona a nivell tècnic i quines avantatge i inconvenients té.

<!-- high five \._./ -->

### The Bitcoin Grouphug

El GroupHug és una solució de _transaction batching_ que permet realitzar crear transaccions conjuntes entre usuaris similars a un coinjoin sense interacció ni coordinació entre els usuaris a canvi d'algunes limitacions.

El principal benefici que aporta aquest model és l'estalvi de comissions. Aquest estalvi pot arribar a ser del 10% aproximadament. En un primer moment pot no semblar un estalvi gaire elevat però en un entorn amb comissions molt altes pot arribar a ser bastant significatiu.

Per tal de cal·lcular l'estalvi que pot donar-nos el GroupHug el que hem de fer és calcular el pès en _virtual bytes_ que te cada camp d'una transacció, l'estalvi sera la relació entre el nombre d'usuaris i el pès, dins d'una transacció, dels camps comuns, és a dir, tot el que no siguin els inputs i els outputs.

Sabem que una transacció segwit P2WPKH amb 1 input i 1 output pesa 109.25 virtual bytes.
Si ens quedem només amb els camps comuns tenim:

| Camp | Mida (Bytes) | Mida (WU)
| ------------ | ------------ | ------------ |
| Tx version        | 4 | 16 |
| Inputs Count      | 1 | 4 |
| Outputs Count     | 1 | 4 |
| nLockTime         | 4 | 16 |
| Segwit Marker     | 1 | 1 |
| Segwit Flag       | 1 | 1 |

Tenim que els camps comuns pesen 12 bytes. Els passem a _Weight Units_ múltiplicant per 4 els camps que no son segwit (Tx Version, Input Count, Output Counti i nLockTime) i obtenim que la mida en WU és de 42 WU. Per passar de WU a virtual bytes dividim per 4 i obtenim que els camps comuns ocupen 10.5 vb.

Saben que els cams comuns representen 10.5/109.25 = ~9.6%.
Per tant amb el GroupHug podem repartir el cost d'aproximadament el 10% de la transacció entre els usuaris.

> :bulb: Per més informació sobre com calcular la mida de les transaccions i perquè s'afirma que una transacció P2WPKH 1in-1out pesa 109.25vb es poden consultar els següents enllaços: [BitcoinStackExchange](https://bitcoin.stackexchange.com/questions/92689/how-is-the-size-of-a-bitcoin-transaction-calculated) i [BitcoinOptech](https://bitcoinops.org/en/tools/calc-size/).


### Funcionament a baix nivell

