+++
title = 'Grouphug - Agrupació de transaccions sense interacció entre usuaris'
date = 2024-05-20
draft = false
tags = ["Sighash", "UTXO management", "PSBT", "Coinjoin", "Layer 1"]
categories = ["Teoria"]
+++

Amb la intenció d'intentar ajudar a fer transaccions de Bitcoin més econòmiques i inspirat en la implementació de [Peach](https://peachbitcoin.com/) feta per en [Czino](https://x.com/capoczino) neix el GroupHug. Un sistema de _transaction batching_ que permet compartir entre usuaris les tarifes d'una transacció sense necesitat de coordinació entre usuaris.

En aquest post s'explica què és el GroupHug, com funciona tècnicament i quins avantatges i inconvenients té.

<!-- high five \._./ -->

### The Bitcoin Grouphug

El GroupHug és una solució de _transaction batching_ que permet crear transaccions conjuntes entre usuaris similars a un coinjoin sense interacció ni coordinació entre els usuaris a canvi d'algunes limitacions.

El principal benefici que aporta aquest model és l'estalvi de comissions. Aquest estalvi pot arribar a ser del 10% aproximadament. En un primer moment pot no semblar un estalvi gaire elevat, però en un entorn amb comissions molt altes pot arribar a ser bastant significatiu.

Per tal de calcular l'estalvi que pot donar-nos el GroupHug el que hem de fer és calcular el pes en _virtual bytes_ que té cada camp d'una transacció, l'estalvi serà la relació entre el nombre d'usuaris i el pes, dins d'una transacció, dels camps comuns, és a dir, tot el que no siguin els inputs i els outputs.

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

Tenim que els camps comuns pesen 12 bytes. Els passem a _Weight Units_ multiplicant per 4 els camps que no són segwit (Tx Version, Input Count, Output Counti i nLockTime) i obtenim que la mida en WU és de 42 WU. Per passar de WU a _virtual bytes_ dividim per 4 i obtenim que els camps comuns ocupen 10.5 vb.

Sabem que els camps comuns representen 10.5/109.25 = ~9.6%.
Per tant, amb el GroupHug podem repartir el cost d'aproximadament el 10% de la transacció entre els usuaris.

> :bulb: Per a més informació sobre com calcular la mida de les transaccions es poden consultar els següents enllaços: [BitcoinStackExchange](https://bitcoin.stackexchange.com/questions/92689/how-is-the-size-of-a-bitcoin-transaction-calculated) i [BitcoinOptech](https://bitcoinops.org/en/tools/calc-size/).


### Funcionament a baix nivell

La principal eina que utilitza el GroupHug per funcionar és el que es coneix com a SigHash. Tot i que no és un terme gaire conegut per l'usuari habitual de Bitcoin el fet és que s'utilitza en cada input i en cada firma d'una transacció.

El SigHash és bàsicament un _flag_ que s'utilitza per indicar què està firmant. La idea que per poder gastar una UTXO s'ha de proporcionar una firma vàlida és ben coneguda, però què es firma exactament? El SigHash ve a indicar això. N'hi ha diferents, però el més habitual és el SigHash `ALL` on en cada firma es firmen TOTS els inputs i TOTS els outputs.

> :bulb: Per a més informació sobre els diferents tipus de SigHash es poden consultar els següents enllaços: [MasteringBitcoin](https://github.com/bitcoinbook/bitcoinbook/blob/6c472dd00b649b18b6ca6bbcc8ba23775619ce08/ch06.asciidoc#signature-hash-types-sighash). Les imatges utilitzades a continuació també són del llibre Mastering Bitcoin.

El SigHash que utilitza el GroupHug és: `SINGLE | ANYONECANPAY`. Aquest està format per dues parts. `SINGLE` significa que la firma aplica a totes les entrades però només a una sortida, concretament a la sortida del seu mateix nivell.

![](/grouphug/sighash_guia.png#center)
![](/grouphug/single.png#center)

Tal com es veu a la figura anterior, amb això aconseguim afegir sortides sense invalidar les firmes prèvies, però no entrades, ja que aquestes van totes firmades i, per tant, la transacció és rígida.

Per tal de poder afegir entrades s'afegeix el `ANYONECANPAY`. Tal com es veu a la figura amb la combinació d'aquests indiquem que les firmes van 1 a 1 inputs amb outputs i, com a resultat, es pot afegir entrades i sortides indiscriminadament sense invalidar les firmes ja fetes.

![](/grouphug/single_anyonecanpay.png#center)


Aquesta doncs és la base de la no-interacció entre usuaris del GroupHug. Com que les firmes van 1 a 1 entre inputs i outputs es poden agrupar transaccions de diferents usuaris **ja firmades** sense invalidar les seves firmes.

Hi ha algun detall més que s'ha de tenir en compte i és que les firmes a part de firmar inputs i outputs firmen alguns camps comuns de la transacció com el `Tx version` i el `LockTime`.

### Avantatges i inconvenients

Els avantatges són fàcils d'identificar, estalviar comissions a l'hora de fer una transacció pagant entre diferents usuaris el cost dels bytes dels camps comuns d'una transacció.

Cal destacar que seria un error entendre el GroupHug com un coinjoin (de l'estil Whirlpool o Wasasbi) o un mixer. Tot i que s'ajuntin entrades i sortides de diferents usuaris aquestes no són barrejades i coincideixen 1 a 1 en l'ordre. A més a més, estan enllaçades per la firma. Cada input d'un usuari està firmant l'output d'aquell mateix usuari. Per tant, el GroupHug **NO ÉS UNA EINA DE PRIVACITAT**.

Com a inconvenients tenim que les transaccions que els usuaris introdueixen al GroupHug han de tenir el mateix nombre d'entrades que de sortides. A més, les transaccions han de complir uns requisits com ara especificar el mateix `Tx version` i mateix `LockTime` per tal de poder agrupar-les sense invalidar les firmes. I s'han de firmar amb el sighash `SINGLE | ANYONECANPAY`, tots aquests requisits són un problema d'UX.

Aquests problemes d'UX es podrien solucionar si el wallet en detectar que la transacció conté el mateix nombre d'inputs que d'outputs, donés l'opció a l'usuari d'enviar-la a un GroupHug i en fer-ho el propi wallet muntes la transacció complint tots els requisits mencionats prèviament.

> :warning: **ALERTA DEV!**\
> *Si algun desenvolupador de wallets llegeix això, ja sap... 😉 Podeu escriure'm al correu que hi ha en aquesta mateixa pàgina, per tal d'implementar-ho, o em podeu trobar per twitter*

### Comunitat

Podeu consultar el codi sobre la meva implementació de GroupHug al repositori de [GitHub](https://github.com/polespinasa/bitcoin-grouphug).
També podeu trobar-lo en funcionament en el servidor de la comunitat de [Barcelona Bitcoin Only](https://x.com/bcnbitcoinonly):
- Mainnet: https://grouphug.bitcoinbarcelona.xyz/
- Testnet3: https://grouphug.testnet.bitcoinbarcelona.xyz/
- Signet: https://grouphug.signet.bitcoinbarcelona.xyz/ (és la [signet de BBO](https://x.com/oomahq/status/1785685345536806986) no l'"oficial")
