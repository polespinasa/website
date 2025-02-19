+++
title = "Com l'UTXO set ens pot ajudar a sincronitzar un node de Bitcoin"
draft = false
date = 2025-02-13
tags = ["Nodes", "BitcoinCore", "Layer 1", "Bitcoin-cli"]
categories = ["Guies"]
about = "Com accelerar la sincronització inicial de la blockchain de Bitcoin en el cas d'haver tingut un node prèviament"
+++

Tots ens hem trobat algun cop que el nostre node de Bitcoin deixa de funcionar, la raspberry falla, el disc mor, o simplement necessitem el disc per altres propòsits i acabem quedant-nos sense node de Bitcoin. En el futur quan volem tornar a muntar un node ens trobem que hem de tornar a passar pel procés de IBD (*initial block download*) per sincronitzar la cadena de blocs. Un procés pesat, que pot durar en molts casos dies i que ens impedeix utilitzar el node i connectar-hi les bitlleteres fins que el node estigui sincronitzat del tot.

En aquest post veurem com podem avançar-nos a aquesta situació i com podem prendre mesures per poder, en un futur, sincronitzar els nostres nodes ràpidament.

> :warning: **CLICK BAIT**\
> *El node no se sincronitza més ràpidament, però sí que et permet utilitzar-lo quasi a l'instant usant sincronització en segon pla!*

## Primeres idees

Hi ha algunes maneres d'accelerar el procés d'IBD sacrificant seguretat no verificant la cadena de blocs. Tot i que algunes no són recomanables, s'expliquen en aquest primer apartat.

### Assume valid

El paràmetre `assumevalid` a Bitcoin Core permet especificar-li al node que fins a certa altura de bloc no vols que es verifiquin les transaccions i assumeix que són vàlides. Per tant, el procés es simplifica a només validar que els blocs estan construïts correctament. Utilitzar aquest paràmetre durant la sincronització inicial accelera moltíssim el procés perquè el principal factor que alenteix la sincronització és la verificació són els scripts.

Tot i això, amb `assumevalid` igualment necessitarem descarregar tota la cadena per poder utilitzar el node. Sí, anirà més ràpid, però la descàrrega l'hem de fer igualment fins a arribar a l'estat actual.

> :bulb: **Informació!**\
> *Bitcoin Core té habilitat `assumevalid` per defecte a una altura de bloc prèvia a la release.*

### Copiar la cadena de blocs

Una altra manera d'accelerar el procés d'IBD és copiar els fitxers d'un node a l'altre. Específicament s'hauria de copiar els fitxers del directori `blocks` i `chainstate`.

Aquest procediment pot tenir alguns problemes. Per començar, descarregar els fitxers d'una font desconeguda no és recomanable. Si has de dur a terme aquest procediment, fes-ho des d'un node que tu controlis. També podrien haver-hi incompatibilitats entre versions del software del node en cas que la manera que creen i emmagatzemen aquests fitxers fos diferent.

Descartem aquesta opció perquè mantenir còpies redundants de la cadena de blocs sencera no és la manera òptima de fer-ho.

## Com ho fem doncs?

Imaginem que poguéssim guardar un *checkpoint* que representés l'estat de la cadena a certa altura de bloc. Quina informació hauria de tenir aquest *checkpoint*? Per verificar transaccions i gastar bitcoins el que necessitem és conèixer les transaccions no gastades en el present. Com podem representar aquestes transaccions no gastades? Amb l'**UTXO set**!

L'**UTXO set** és una base de dades que representa l'estat actual de les sortides de transacció no gastades. Amb l'**UTXO set** podem verificar que una transacció és vàlida sense mirar la cadena, ja que les entrades d'aquestes han de gastar *unspent transaction outputs* (UTXOs) i crear-ne de nous.

Aquest altre post explica què és i perquè serveix l'[**UTXO set**]({{< relref "utxo_set.md" >}})

### Com utilitzem l'UTXO set per sincronitzar el node

Si tenim l'estat actual de la cadena, és a dir, les monedes (UTXOs) que es poden gastar i tenim el hash del bloc a aquella altura podem connectar la nostra bitlletera. Això funciona ja que tots els bitcoins que tenim disponibles per gastar són representats a l'**UTXO set** i, mentrestant, els blocs previs es podran anar descarregant en segon pla sense influir en la capacitat de les wallets per mostrar el balanç i gastar fons.

### Carregar còpia de l'UTXO set

Suposem que tenim un backup de l'**UTXO set** en un fitxer: `utxo.dat`. Aquest backup de l'**UTXO set** està fet en el bloc 840.000, per tant, quan l'importem el nostre node s'haurà d'actualitzar a l'altura 840.000 a l'instant.
Per fer-ho podem utilitzar la [comanda que introdueix Bitcoin Core a la versió 26.0 loadtxoutset](https://bitcoincore.org/en/doc/28.0.0/rpc/blockchain/loadtxoutset/).

Comencem des del principi. Iniciem el node de Bitcoin executant, com és habitual, el binari `bitcoind`. Veurem que al principi realitza una presincronització dels *headers* i després els sincronitza. Hem d'esperar que la sincronització dels *headers* finalitzi.

Presincronització:
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


Sincronització:
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

Un cop els *headers* estan sincronitzats, si executem `bitcoin-cli getblockchaininfo` veurem que comença a sincronitzar els blocs normal i corrent:
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

Un cop tenim el node ja corrent i sincronitzant podem importar l'**UTXO set** amb la instrucció `bitcoin-cli -rpcclienttimout=0 loadtxoutset utxo.dat`. On `utxo.dat` és el nostre backup de l'**UTXO set**. (Recomano col·locar el fitxer `utxo.dat` a dins el directori on estiguis guardant les dades del node, per defecte és `~\.bitcoin`).

> :warning: **Alerta**\
> *És important especificar `-rpcclienttimout=0` per evitar que ens tanqui la comunicació per timeout, ja que el procés pot trigar uns minuts*

És normal que la terminal quedi penjada sense actuar tal que:
![](/assumetxoutset/txoutset_timout.png#center)

Mentre la comanda sembla que queda penjada es pot veure en els logs del `bitcoind` com el node continua sincronitzant-se de manera normal i, eventualment, es poden observar logs on mostra que està carregant les *coins* de l'**UTXO set**.
![](/assumetxoutset/load_coins.png#center)

Un cop acabat veurem que la comanda RPC confirma que hem importat les dades:
```bash
bitcoin-cli -rpcclienttimeout=0 loadtxoutset utxo.dat 
{
  "coins_loaded": 176948713,
  "tip_hash": "0000000000000000000320283a032748cef8227873ff4872689bf23f1cda83a5",
  "base_height": 840000,
  "path": "/home/sliv3r/Escritorio/blockchain/utxo.dat"
}
```

Si observem els logs de `bitcoind`, podrem veure com ha actualitzat el `chainstate` (l'estat de la cadena) al bloc 840.000:
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

I si executem la comanda `bitcoin-cli getblockchaininfo` podrem veure com efectivament tenim el node sincronitzat al bloc 840.000:

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

Tornem a mirar els logs de `bitcoind` i veiem que ara està sincronitzant els blocs que hi ha entre el 840.000 i el tip de la cadena:
> :bulb: S'anomena tip al bloc actual de la cadena, és a dir a l'últim bloc de la cadena amb més proof of work.
```
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000001caffc952e7de3b1c8b2f2d044e8e69955000ca6fb9e4 height=840436 version=0x20e00000 log2_work=94.879663 tx=993054314 date='2024-04-23T01:53:38Z' progress=0.849873 cache=458.5MiB(2943813txo)
2025-02-18T14:58:36Z UpdateTip: new best=000000000000000000014797e1aa0867732005d6cb11fad392e35fbc81f552bd height=840437 version=0x20000000 log2_work=94.879678 tx=993060398 date='2024-04-23T02:05:34Z' progress=0.849878 cache=459.1MiB(2947040txo)
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000002a0658cf7d5af3fadb3469df23e97de849a32991c0545 height=840438 version=0x3fffe000 log2_work=94.879693 tx=993066120 date='2024-04-23T02:13:23Z' progress=0.849883 cache=459.8MiB(2951620txo)
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000002349473de1b8f66f12b65cdeb870c211927865c856e41 height=840439 version=0x3fff0000 log2_work=94.879707 tx=993071846 date='2024-04-23T02:16:31Z' progress=0.849888 cache=460.5MiB(2956234txo)
2025-02-18T14:58:36Z UpdateTip: new best=00000000000000000000c6b9fd50913cf6d3d34536c06c1a029ba6c844cc8033 height=840440 version=0x20a00000 log2_work=94.879722 tx=993077310 date='2024-04-23T02:17:04Z' progress=0.849893 cache=461.2MiB(2961405txo)
2025-02-18T14:58:36Z UpdateTip: new best=000000000000000000033c2f66f551e44aea785f91a902b321c38703a158ed3a height=840441 version=0x20a00000 log2_work=94.879737 tx=993083167 date='2024-04-23T02:20:03Z' progress=0.849898 cache=461.9MiB(2965605txo)
2025-02-18T14
```

Un cop el node acaba de sincronitzar des del bloc 840.000 fins al tip actual, en aquest cas el bloc 884365, podem veure com comença a sincronitzar el node des de 0. Es pot identificar fàcilment, ja que en els logs es marca com a *background validation*.

```
2025-02-18T20:18:41Z [background validation] UpdateTip: new best=000000000000001db5a1515a5f8534c941b1628f60466e6b709b3b320254afff height=208000 version=0x00000001 log2_work=69.023342 tx=8883319 date='2012-11-15T05:17:32Z' progress=0.007602 cache=58.2MiB(460376txo)
2025-02-18T20:19:04Z [background validation] UpdateTip: new best=000000000000048b95347e83192f69cf0366076336c639f9b7228e9ba171342e height=210000 version=0x00000002 log2_work=69.091539 tx=9344662 date='2012-11-28T15:24:38Z' progress=0.007996 cache=94.6MiB(728202txo)
2025-02-18T20:19:40Z [background validation] UpdateTip: new best=00000000000003d906e4131c39f7655b72df40146d2967f5d75113a09610de61 height=212000 version=0x00000002 log2_work=69.157547 tx=9819223 date='2012-12-13T01:33:36Z' progress=0.008403 cache=117.6MiB(927414txo)
2025-02-18T20:19:41Z Saw new header hash=000000000000000000019940b918fed6453c7afa80e03f1d837d1fdd8b31a8b9 height=884366
2025-02-18T20:19:45Z UpdateTip: new best=000000000000000000019940b918fed6453c7afa80e03f1d837d1fdd8b31a8b9 height=884366 version=0x2002a000 log2_work=95.452620 tx=1156361979 date='2025-02-18T20:19:40Z' progress=1.000000 cache=2.7MiB(19061txo)
```
Fixant-nos en els logs anteriors podem veure que, tot i que està verificant blocs molt antics com el 210000, quan apareix un bloc nou, el 884366 en aquest cas, de seguida actualitza el tip de la seva cadena i posteriorment continua amb la verificació de blocs antics.


## Conclusions i precaucions

Amb aquest procés el node arriba a un estat on és usable per consultar el balanç i crear transaccions de manera molt més ràpida que no pas sincronitzant tota la cadena. En un portàtil normal, tot aquest procés no m'ha pres més de 2 hores que ja és un avanç mot gran respecte als dies que pot arribar a trigar una sincronització normal.

> :warning: **Alerta**\
> *És molt important no descarregar l'__UTXO set__ d'una font desconeguda i amb la que no confiem. El perill és molt elevat, ja que una representació falsa de l'__UTXO set__ pot fer-te creure que t'han pagat quan no és cert. __Et poden enganyar i robar__.*

Bitcoin Core intenta prevenir això limitant la importació de l'**UTXO set** al bloc 840.000 i comprovant que l'**UTXO set** és el correcte i no ha estat manipulat. Saltar-se aquesta comprovació no és difícil i permet a usuaris més experimentats accelerar encara més aquest procés. En aquest post no s'explicarà com fer-ho per tal de no exposar als lectors a un possible atac.

**RECORDA ---> DON'T TRUST, VERIFY. IF YOU DON'T RUN YOUR OWN NODE YOU ARE NOT USING BITCOIN WITHOUT A "TRUSTED" MIDLEMAN**


### POL PERÒ NO M'HAS ENSENYAT A CREAR EL BACKUP!!

Cert, no ho he explicat. Ho explicaré en un futur. La funcionalitat per fer-ho encara no està a cap *release* oficial de Bitcoin Core, està en el codi per la següent *release*. Com que no vull incentivar a la gent a córrer software que no s'ha publicat "oficialment" no publicaré les instruccions de com fer-ho servir fins que surti en una versió oficial.

![](/assumetxoutset/engañado.png#center)