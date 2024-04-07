+++
title = "Com fer una transacció amb UTXOs de wallets diferents amb Sparrow"
draft = false
date = 2024-04-07
tags = ["Wallets", "UTXO management", "PSBT", "Sparrow", "Layer 1", "Bitcoin-cli"]
categories = ["Guies"]
+++

Suposem que en Bob i la Alice comparteixen pis i volen comprar un televisor que val 2 bitcoins pagant la meitat cada un.
Per fer-ho en Bob podria enviar 1 bitcoin a la Alice perquè ella fes el pagament del televisor, però com que les comissions de xarxa són bastant elevades, decideixen crear una transacció conjunta on hi hagi dos inputs, un de l'Alice i un del Bob.

> :bulb: *Els exemples pràctics del post estan fets en testnet. Per tant, si es vol executar les comandes del Bob i l'Alice amb els mateixos valors o buscar les transaccions a un explorador s'ha de fer des de les versions de testnet corresponents.*

### Crear la PSBT

El primer que fan és crear una transacció parcialment firmada (PSBT) on especifiquen les UTXO que vol gastar cada un i la sortida que volen crear.

Les dades per l'entrada de la PSBT són les dues UTXO de 1.000005 bitcoins cada una:

UTXO de l'Alice: `73c13b277a1b176fd2b20324027fff9244949fc5dc913aa98c68bddf246260e5:0`\
UTXO del Bob: `7797817c9b879c3333e1c7d5eedc70e6aa0ba1a2c6d098c94ce5e2671f4c9d5e:1`

Les dades per la sortida són l'adreça de destí i la quantitat de bitcoins que hi volen enviar:

Adreça destí: `tb1qjdntspzahv748vasmnfl62etfz9y9hjgmvg6gw`\
Quantitat de bitcoins: `1 btc`

Com que utilitzen [Sparrow](https://sparrowwallet.com/) com a wallet i aquesta no permet la creació de PSBTs han d'utilitzar una altra eina. Per fer-ho faran servir el seu propi node de Bitcoin i la seva interfície de comandes ([bitcoin-cli](https://developer.bitcoin.org/reference/rpc/index.html)).

Per crear la PSBT fan servir bitcoin-cli i la comanda ["*createpsbt*"](https://developer.bitcoin.org/reference/rpc/createpsbt.html).\
Aquesta pren dos arguments obligatoris els inputs i els outputs. Per l'input s'ha d'especificar l'id de la transacció i el número de l'output dins de la transacció. Per l'output s'ha d'especificar l'adreça de destí i la quantitat de bitcoins que es vol enviar.
```bash
bitcoin-cli createpsbt '[{"txid": "txId", "vout": nOut}]' '["adreça de destí": "núm btc"]'
```
> :warning: **ALERTA!**\
> *La diferència entre el valor de les entrades i el valor de la sortida es paga en forma de comissió. Si es vol retornar els diners restants, s'ha d'afegir una adreça de canvi i donar-li un valor!*


La comanda que executen l'Alice i en Bob queda tal que així:
```bash
bitcoin-cli createpsbt '[{"txid": "73c13b277a1b176fd2b20324027fff9244949fc5dc913aa98c68bddf246260e5", "vout": 0}, {"txid": "7797817c9b879c3333e1c7d5eedc70e6aa0ba1a2c6d098c94ce5e2671f4c9d5e", "vout": 1}]' '[{"tb1qjdntspzahv748vasmnfl62etfz9y9hjgmvg6gw": 2}]'
```
Output:
```
cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAAAA
```
L'output que obtenen és la transacció en raw i codificada en base64. Però aquesta transacció encara no està llesta per ser firmada. Necessita que se li actualitzi la informació dels inputs i els outputs segwitt. Per tal d'acabar de muntar correctament la transacció en Bob i l'Alice utilitzen la comanda ["*utxoupdatepsbt*"](https://developer.bitcoin.org/reference/rpc/utxoupdatepsbt.html).
Aquesta comanda pren per com a únic argument una transacció en raw codificada en base64, exactament com la que ens genera la comanda "*createpsbt*".
```bash
bitcoin-cli utxoupdatepsbt "transacció en raw i base64"
```
Per tant, la comanda que executen l'Alice i en Bob queda de la següent manera:
```bash
bitcoin-cli utxoupdatepsbt "cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAAAA"
```
Output:
```
cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocwABAR/04vUFAAAAABYAFKnH89dqqyObFvPlDONKPDSRm78cAAA=
```
L'output que obtenen és la mateixa transacció en raw i codificada en base64 però aquest cop ja llesta per ser firmada.
Es pot comprovar l'estat de la PSBT amb la comanda ["*analyzepsbt*"](https://developer.bitcoin.org/reference/rpc/analyzepsbt.html). Aquesta analitza la PSBT i dona informació sobre l'estat d'aquesta.
La comanda només pren com a argument una PSBT en raw i base64:
```bash
bitcoin-cli analyzepsbt "transacció en raw i base64"
```
En Bob i l'Alice poden comprovar l'estat de la seva PSBT amb la següent comanda:
```bash
bitcoin-cli analyzepsbt "cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocwABAR/04vUFAAAAABYAFKnH89dqqyObFvPlDONKPDSRm78cAAA="
```
Output:
```json
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "updater",
      "missing": {
        "pubkeys": [
          "b0713d0cbeb8dbb47f13bbb516ad853b9d4e6873"
        ]
      }
    },
    {
      "has_utxo": true,
      "is_final": false,
      "next": "updater",
      "missing": {
        "pubkeys": [
          "a9c7f3d76aab239b16f3e50ce34a3c34919bbf1c"
        ]
      }
    }
  ],
  "fee": 0.00001000,
  "next": "updater"
}
```
En el JSON que ens proporciona la sortida podem veure com falten les dues firmes, tant la de l'Alice com la del Bob.

> :bulb: **Idea!**\
> *La comanda anterior també ens permet veure quants bitcoins paga la transacció en forma de comissions, és bona idea revisar que no estem pagant més del compte!*

### Firmar la PSBT

La primera a firmar la transacció serà l'Alice. Per fer-ho obra la seva wallet a Sparrow i carrega la transacció obtinguda de la comanda "*utxoupdatepsbt*" des de text:

![](/psbt_multi_utxo_sparrow/openTx.png#center)
![](/psbt_multi_utxo_sparrow/open_from_text.png#center)

Un cop carregada la transacció es pot veure com el propi Sparrow detecta que la primera entrada pertany a l'Alice:
![](/psbt_multi_utxo_sparrow/tx_pre_sign_alice.png#center)

Per tal de firmar-la l'Alice prem "*Finalize Transaction for Signing*" i a continuació "*Sign*".
Un cop firmada l'Alice es guarda la transacció prement l'opció de "Save Transaction". Això li genera un fitxer "*.psbt*".

Com que té curiositat per saber si s'ha fet correctament torna a executar la comanda "analyzepsbt". Aquesta pren com a argument una transacció en RAW i codificada en Base64, per tal de mostrar la transacció del fitxer .psbt en el format necessari l'Alice utilitza la comanda:

```bash
cat alice.psbt | base64 -w 0
```
Output:
```
cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocyICAqMy+J7K6eLwFZ82KfZ8NEOYP/fT2XJBP7fZKzgTHebMRzBEAiAfOG5ztzUxczR0VOE/RyEHFzE+7WzxURU2M3uaMneq2wIgdNMkWI40nt+IB+LFq8vJ+/s3JqDCk/0IB51Bj/vdtFcBAAEBH/Ti9QUAAAAAFgAUqcfz12qrI5sW8+UM40o8NJGbvxwAAA==
```

Ara ja sí que pot executar la comanda "*analyzepsbt*" amb la transacció codificada correctament:
```bash
bitcoin-cli analyzepsbt "cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocyICAqMy+J7K6eLwFZ82KfZ8NEOYP/fT2XJBP7fZKzgTHebMRzBEAiAfOG5ztzUxczR0VOE/RyEHFzE+7WzxURU2M3uaMneq2wIgdNMkWI40nt+IB+LFq8vJ+/s3JqDCk/0IB51Bj/vdtFcBAAEBH/Ti9QUAAAAAFgAUqcfz12qrI5sW8+UM40o8NJGbvxwAAA=="
```
Output:
```json
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "finalizer"
    },
    {
      "has_utxo": true,
      "is_final": false,
      "next": "updater",
      "missing": {
        "pubkeys": [
          "a9c7f3d76aab239b16f3e50ce34a3c34919bbf1c"
        ]
      }
    }
  ],
  "fee": 0.00001000,
  "next": "updater"
}
```
Ara l'Alice té la certesa que ha firmat, ja que la sortida de la comanda mostra que només falta la firma d'en Bob.

En Bob realitza els mateixos passos que ha fet l'Alice per firmar la transacció.
> :warning: **Alerta!**\
> *En Bob repeteix els passos amb la transacció obtinguda en la comanda "utxoupdatepsbt" no amb la transacció firmada de l'Alice!*

En acabat executa la comanda "*analyzepsbt*" també per assegurar-se que ha firmat correctament:
```bash
bitcoin-cli analyzepsbt "cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocwABAR/04vUFAAAAABYAFKnH89dqqyObFvPlDONKPDSRm78cIgIDDHkSKRVSKLVTwBM8zclLbfzoU0ax431MOeTO7jhNgQJHMEQCIHu7QJ227MnOCoWIgkfKkFpiBlUEfm/Tzs66+wSti5LnAiB767C0BufcXueZR/muH+DeFVSGTE+Hbjt8sgvlyUDGZAEAAA=="
```
Output:
```json
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "updater",
      "missing": {
        "pubkeys": [
          "b0713d0cbeb8dbb47f13bbb516ad853b9d4e6873"
        ]
      }
    },
    {
      "has_utxo": true,
      "is_final": false,
      "next": "finalizer"
    }
  ],
  "fee": 0.00001000,
  "next": "updater"
}
```
En Bob pot veure que la seva firma s'ha afegit correctament i li marca com a pendent la firma de l'Alice.

### Fusionar les PSBT i retransmetre la transacció final

Arribats a aquest punt en Bob i l'Alice tenen cada un la PSBT firmada per separat. Només els falta fusionar-les per tal de tenir la transacció final i poder-la retransmetre a la xarxa.

Per fusionar-les utilitzen la comanda ["*combinepsbt*"](https://developer.bitcoin.org/reference/rpc/combinepsbt.html).
Aquesta pren per arguments les PSBT en una llista de la següent manera:
```bash
bitcoin-cli combinepsbt '["psbt_alice", "psbt_bob"]'
```
La comanda que executen és la següent:
```bash
bitcoin-cli combinepsbt '["cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocyICAqMy+J7K6eLwFZ82KfZ8NEOYP/fT2XJBP7fZKzgTHebMRzBEAiAfOG5ztzUxczR0VOE/RyEHFzE+7WzxURU2M3uaMneq2wIgdNMkWI40nt+IB+LFq8vJ+/s3JqDCk/0IB51Bj/vdtFcBAAEBH/Ti9QUAAAAAFgAUqcfz12qrI5sW8+UM40o8NJGbvxwAAA==", "cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocwABAR/04vUFAAAAABYAFKnH89dqqyObFvPlDONKPDSRm78cIgIDDHkSKRVSKLVTwBM8zclLbfzoU0ax431MOeTO7jhNgQJHMEQCIHu7QJ227MnOCoWIgkfKkFpiBlUEfm/Tzs66+wSti5LnAiB767C0BufcXueZR/muH+DeFVSGTE+Hbjt8sgvlyUDGZAEAAA=="]'
```
Output:
```
cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocyICAqMy+J7K6eLwFZ82KfZ8NEOYP/fT2XJBP7fZKzgTHebMRzBEAiAfOG5ztzUxczR0VOE/RyEHFzE+7WzxURU2M3uaMneq2wIgdNMkWI40nt+IB+LFq8vJ+/s3JqDCk/0IB51Bj/vdtFcBAAEBH/Ti9QUAAAAAFgAUqcfz12qrI5sW8+UM40o8NJGbvxwiAgMMeRIpFVIotVPAEzzNyUtt/OhTRrHjfUw55M7uOE2BAkcwRAIge7tAnbbsyc4KhYiCR8qQWmIGVQR+b9POzrr7BK2LkucCIHvrsLQG59xe55lH+a4f4N4VVIZMT4duO3yyC+XJQMZkAQAA
```

La sortida de la comanda és la PSBT agregada dels dos. Poden fer una última comprovació amb la comanda "*analyzepsbt*". La sortida de la comanda que obtenen és la següent:
```bash
bitcoin-cli analyzepsbt "cHNidP8BAHsCAAAAAuVgYiTfvWiMqTqR3MWflESS/38CJAOy0m8XG3onO8FzAAAAAAD9////Xp1MH2fi5UzJmNDGoqELquZw3O7Vx+EzM5yHm3yBl3cBAAAAAP3///8BAMLrCwAAAAAWABSTZrgEXbs9U7Ow3NP9KytIikLeSAAAAAAAAQEf9OL1BQAAAAAWABSwcT0MvrjbtH8Tu7UWrYU7nU5ocyICAqMy+J7K6eLwFZ82KfZ8NEOYP/fT2XJBP7fZKzgTHebMRzBEAiAfOG5ztzUxczR0VOE/RyEHFzE+7WzxURU2M3uaMneq2wIgdNMkWI40nt+IB+LFq8vJ+/s3JqDCk/0IB51Bj/vdtFcBAAEBH/Ti9QUAAAAAFgAUqcfz12qrI5sW8+UM40o8NJGbvxwiAgMMeRIpFVIotVPAEzzNyUtt/OhTRrHjfUw55M7uOE2BAkcwRAIge7tAnbbsyc4KhYiCR8qQWmIGVQR+b9POzrr7BK2LkucCIHvrsLQG59xe55lH+a4f4N4VVIZMT4duO3yyC+XJQMZkAQAA"
```
```json
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "finalizer"
    },
    {
      "has_utxo": true,
      "is_final": false,
      "next": "finalizer"
    }
  ],
  "estimated_vsize": 177,
  "estimated_feerate": 0.00005649,
  "fee": 0.00001000,
  "next": "finalizer"
}
```
Comproven així que no falta cap de les dues firmes.
A partir d'aquest punt qualsevol dels dos pot retransmetre la transacció carregant-la a l'Sparrow des de text com s'ha fet en els passos anteriors i prenent l'opció de "*Broadcast*"
![](/psbt_multi_utxo_sparrow/broadcast.png#center)

Podem comprovar a qualsevol explorador de blocs que la [transacció](https://mempool.space/testnet/tx/c659c7cb31733f153f04c1fbffeec91b99a96f2e289f47852374d0101e303e50) s'ha enviat correctament i paga els 2 bitcoins pel televisor:

id de la transacció: `c659c7cb31733f153f04c1fbffeec91b99a96f2e289f47852374d0101e303e50`
![](/psbt_multi_utxo_sparrow/mempool_space.png#center)

La transacció ha sigut enviada correctament i en Bob i l'Alice han pogut pagar el televisor amb una transacció conjunta!!
![](/psbt_multi_utxo_sparrow/tv.jpeg#center)