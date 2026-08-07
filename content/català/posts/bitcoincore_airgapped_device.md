+++
title = 'Bitcoin Core com a dispositiu de firma air-gapped'
date = 2026-08-14
draft = false
tags = ["Air-gapped", "Hardware wallet", "Bitcoin Core", "Wallet", "Descriptors"]
categories = ["Guies"]
+++


> :bulb: *La creació d'aquesta guía esta inspirada en el [tutorial de youtube](https://www.youtube.com/watch?v=B5X2LkrVuBM&t=783s) fet per [402 Payment Required](https://x.com/402PaymentReq). Recomano donar-li un cop d'ull a la guía i a la resta de vídeos que té.*

Quan parlem d'autocostòdia sovint parlem de minimització de riscos. Riscos n'hi ha molts i diversos, des de riscos on els fons son robats fins a riscos on els fons son perduts per el propi usuari, riscos a l'hora de fer copies de seguretat, etc.

Dins l'ecosistema tot el mon coneix la frase "_no confiis, verifica_", de l'anglès "_don't trust, verify_", però realment la majoria dels usuaris no verifiquen sino que confien que les eines que utilitzen han sigut verificades per persones que normalment no coneixen però que tenen, o se suposa que tenen, certa trajectòria i reputació.
No obstant això no sempre és així, i ha quedat demostrat amb el recent incident de ColdCard, on un bug en el codi d'una de les hardware wallets amb més reputació de l'escosistema, ha provocat que actors maliciosos puguesin robar els fons de persones que havien seguit casi TOTES les bones pràctiques de seguretat que se solen recomenar.

Si bé és veritat que el codi de ColdCard era públic i es podia auditar (Source Available), la falta d'un programa de recompenses de bugbounty o el fet de no ser de codi obert segurament han fet que aquest bug no es detectés abans. Podem entendre la llicència Source Available, com una llicència que permet als actors maliciosos estudiar en codi per trobar vulnerabilitats, però a la vegada que desincentiva que actors honestos revisin i auditin el codi, ja que no el poden fer servir.

Aleshores què pot fer un usuari que no enten el codi, i per tant no pot verificar, però que, com hem vist, tampoc pot confiar cegament en que el software que utilitza és correcte i no te bugs?

Aquí hi ha tres opcions principals:
1. Per cada software que es fa servir buscar quin software, dels que cumpleix els requisits que es necesiten, és més obert i té més comunitat de desenvolupadors. Reduint així el risc que un bug com el de ColdCard passi desaparcebut.
2. No posar tots els ous a la mateixa cistella quan es fan servir softwares crítics. Per exemple en el cas de les hardware wallets, en comptes d'utilitzar una bitlletera single-sig amb un model de hardware wallet, crear una bitlletera multi-sig on cada firma és gestionada per una hardware wallet diferent. Si una hardware wallet conté un bug com el de ColdCard, els fons estaran assegurats per la resta de bitlleteres.
3. A diferència del punt 2, una altre opció és posar tots els ous en una única cistella. En aquest cas fer-ho tot amb un sol software en el que es confia.

En aquest artícle ens centrarem en la opció 3 per definir un entorn per utilitzar Bitcoin amb una hardware wallet, o almenys, amb un dispositiu airgapped.

## Bitcoin Core com a unic software relacionat amb Bitcoin

Normalment els entorns més habituals per a Bitcoiners consisteixen en tenir corrent, un node, un servidor d'electrum, una hot-wallet (en funció de watch-only wallet) i una hardware wallet. Evidentment poden haver-hi modificacions, com ara precindir del servidor d'electrum o del node, d'entre altres.

En l'entorn definit es depen de quatre softwares diferents i per tant són quatre punts de confiança i quatre punts crítics on un bug pot ser fatal.
La idea principal d'aquest pos és explicar com reduir aquests riscos i aquesta confiança a un sol software. Es busca utilitzar el mateix software per el node, la watch-only wallet i la wallet air-gapped.

Com que també es vol que el software que s'esculleixi estigui ben auditat i hi hagi molts ulls sobre el codi per minimitzar el risc de bugs crítics, el software escollit és Bitcoin Core.

En els següents apartats s'explicarà com es pot muntar un entorn on es te un node de Bitcoin Core corrent i una wallet de Bitcoin Core aillada completament que actuara com a substitut a una hardware wallet.

Els requisits són:
- Un ordinador conectat a internet.
- Un ordinador que NOMÉS podras utilitzar per firmar transaccions, no pot fer cap altre ús i no es podrà connectar MAI més a internet.
- Tres USBs o microSDs completament nous.

### Requisits dels ordinadors 

La guía no especifica quins requisits han de tenir els ordinadors. Els requisits per correr Bitcoin Core són mínims i per tant qualsevol ordinador amb una mica d'espai al disc (~15GB) hauria de ser suficient.

Els requisits de seguretat ja depenen del nivell de paranoia de cada un. Obviament per l'ordinador air-gapped seria interessant poder tallar la conexió a Internet de manera física, per exemple trancant i desconectant l'antena física.

> :warning: **ALERTA!**\
> *Per tenir un dispositiu realment air-gapped, l'antena de wifi i la de Bluetooth s'han de trencar o deshabilitar d'alguna manera, la majoria d'aquests xips corren software propietari i no podem saber realment què estan fent. L'única manera d'estar segur al 100% que el dispositiu està aïllat d'Internet és trencant-les o deshabilitant-les a nivell de harware.*

Poden haver-hi altres preocupacions com, com em fio de la meva BIOS? Puc confiar en el procesador? d'entre altres. Tots aquests temes tenen solució i hi ha alternatives per tot, però això queda fora de l'abast d'aquesta guia i recomano que, si us interessen i/o preocupen aquests temes, feu recerca sobre ells.

> :bulb: *Si algú li interesa comprar un ordinador amb uns requisits de privacitat i seguretat més elevats pot donar-li un cop d'ull a la web de [SilkPad](https://silkpad.net/) feta per [ChavoTheDruid](https://x.com/ChavoGnuGrowers). Recomano que qualsevol pregunta sobre aquesta tema li feu a ell, ja que te molts més coneixaments i experiència que jo.*

## Guía de com utilitzar Bitcoin Core com únic software relacionat amb Bitcoin i mantenir un setup air-gapped

> :warning: **ALERTA!**\
> *Tot i que l'entorn que s'acaba generant és bastant segur, la practicitat del mateix depen dels coneixaments dels úsuaris. No recomanaria aquest entorn per usuaris que no tenen unes mínimes nocions de seguretat i informàtica. Minimitzant alguns riscos se n'introdueixen d'altres, en aquest cas, el risc d'un error humà.*

Tot el proces que s'explicarà a continuació en detall es pot reduir en 6 simples passos:

1. Instalar tails a un USB nou i arrancar el sistema operatiu a l'ordinador aillat d'Internet.
2. A l'ordinador connectat a Internet descargar Bitcoin Core i copiar-lo dins d'un dels USBs nous.
3. Utilitzant el programa copiat a l'USB, instalar Bitcoin Core a l'ordinador aillat d'Internet i generar una wallet nova amb les claus privades.
4. Exportar els descriptors públics de la wallet i copiar-los en un altre dels USBs nous.
5. A l'ordinador connectat a Internet instalar Bitcoin Core, sincronitzar el node i importar els descriptors públics.
6. A l'hora de fer transaccions generar PSBTs a l'ordinador conectat a Internet i firmar-les des de l'ordinador desconnectat. Per importar i exportar les PSBTs es pot fer amb USBs o microSDs completament noves o amb codis QR.

> :bulb: *Nota prèvia: en el pas 4 i 5 s'utilitza una funcionalitat que encara no està disponible a la última versió de Bitcoin Core (v31), en principi a partir de la versió v32 ja es podra fer servir. Tot i això tot aquest procediment es pot seguir igualment però es complica una mica. Recomano mirar la part 4 del [tutorial de youtube](https://www.youtube.com/watch?v=B5X2LkrVuBM&t=783s) fet per [402 Payment Required](https://x.com/402PaymentReq).*

> :warning: **ALERTA!**\
> *Durant to el procés es connecten diversos USBs a l'ordinador air-gapped, és important destruir-los un cop s'han utilitzat, ja que si hi hagués malware instal·lat podria intentar copiar el fitxer la wallet, intentar exportar les claus privades i filtrar informació. El principi d'air-gapped no s'hauria de violar MAI!!*

### Pas 1. Instalar Tails a un USB nou

Si ja saps com instalar la imatge de tails a un USB, pots saltar al següent pas.

El primer pas és descargar la imatge de tails de la [web oficial](https://tails.net/install/download/index.en.html). Just amb el fitxer `.img` idealment s'haurien de descarregar les firmes PGP i les claus públiques corresponents, aquestes es poden trobar a la mateixa web.

Un cop tot descarregat s'hauria de verificar que el fitxer `.img` és el correcte. Per fer-ho es verifiquen les firmes, si ets un usuari acostumat a la linia de comandes pots utilizar la comanda:

```bash
gpg --verify tails-xxx-x.x.img.sig
```

![](/bitcoincore_offline_signing/verify_tails_image_terminal.png#center)

Si prefereixes no utilitzar la linia de comandes pots utilitzar alguna interficie gràfica per fer-ho. Per exemple [Sparrow](https://www.sparrowwallet.com/) integra una eina per verificar firmes PGP:

![](/bitcoincore_offline_signing/verify_tails_image_sparrow.png#center)

Un cop es verifica que les signatures son vàlides es pot procedir a escriure la imatge a un USB, per fer-ho s'ha d'utilitzar algún programa que t'ho permeti. En el meu cas com que tinc un entorn KDE, aquest ve amb un programa per defecte, però n'hi ha molts que es poden trobar per internet.

<div style="display: flex; gap: 10px; justify-content: center; flex-wrap: wrap;">
  <img src="/bitcoincore_offline_signing/usb_loader1.png" style="max-width: 45%; min-width: 300px;" />
  <img src="/bitcoincore_offline_signing/usb_loader2.png" style="max-width: 45%; min-width: 300px;" />
</div>

Un cop l'USB esta a punt amb la imatge gravada s'ha de conectar a l'ordinador que tindrem desconectat d'Internet i que actuara com a "hardware wallet".

S'ha d'arrencar l'ordinador i a la BIOS seleccionar que arranqui amb l'USB en comptes d'arrancar amb el disc normal. Com fer això dependra de l'ordinador, així que no donare detalls al respecte.

Si tot va bé t'hauries de trobar en una pantalla similar a aquesta:

![](/bitcoincore_offline_signing/welcome_to_tiles.jpg#center)

És important seleccionar l'opció "_Create Persistent Storage_" i a l'apartat de "_Additional Settings_" activar l'Offline Mode i deshabilitar el navegador. L'objectiu és tallar a nivell de sistema operatiu l'accés a Internet del dispositiu.

Un cop es premi el boto "**Start Tails**" apareix una pantalla per iniciar el "_Persistent Storage_". Tails elimina totes les dades cada cop que s'apaga l'ordinador. 
![](/bitcoincore_offline_signing/persistent_storage1.png#center)

El _Persistent Storage_ és una partició encriptada que Tails crea a l'USB i és l'única informació que es manté en apagar l'ordinador. Alla iguardarem el binari de Bitcoin Core i les claus privades de la wallet.

> :warning: **ALERTA!**\
> *La contrasenya que Tails demana és la que s'utilitza per encriptar la partició, és important no oblidar aquesta contrasenya ja que sino el contingut no es pot recueprar. També és important posar una contrasenya suficientment complicada perquè un atacant no la pugui esbrinar!*

Obviament "test" no és una bona contrasenya...

![](/bitcoincore_offline_signing/persistent_storage2.png#center)

Un cop creat el persistent storage, Tails pregunta què s'hi vol emmagatzemar, s'ha d'activar la carpeta `Persistent Folder` ja que alla s'hi guardara el binari de Bitcoin Core i la wallet.

Addicionalment es pot activar el `GnuPG` on s'hi poden emmagatzemar claus publiques per verificar binaris. Això pot ser útil si es vol actualitzar el binari de Bitcoin Core en un futur.

![](/bitcoincore_offline_signing/persistent_sotrage3.png#center)

### Pas 2. Verificar Bitcoin Core i portar-lo al dispositiu off-line

Un cop tenim l'ordinador air-gapped amb tails necesitem instalar-hi Bitcoin Core, per fer-ho descarreguem el Binari des de la web [bitcoincore.org](bitcoincore.org) o el repositori de [github](github.com/bitcoin/bitcoin). Junt amb el binari descarreguem les firmes i, si no les tenim ja importades, les claus publiques dels contribuidors de Bitcoin Core.

> :bulb: *Les claus dels contribuidors de Bitcoin Core es poden trobar a https://github.com/bitcoin-core/guix.sigs/tree/main/builder-keys*

Els fitxers que s'haurien de tenir són:
- Fitxer comprimit amb els binaris de bitcoin.
- Fitxer amb els hashos dels fitxers &rarr; `SHA256SUMS`
- Fitxer amb les signatures del fitxer `SHA256SUMS` &rarr; `SHA256SUMS.asc`
- Carpeta amb les claus públiques &rarr; `guix.sigs/builder-keys`

Un cop els tres fitxers són descarregats s'han d'importar les claus públiques dels firmants.
![](/bitcoincore_offline_signing/import_core_pgp_keys.png#center)

Comparar que el hash del fitxer comprimit coincideix amb el hash que hi ha definit a `SHA256SUMS`.

![](/bitcoincore_offline_signing/verify_core_hash.png#center)

I a continuació verificar que les signatures són correctes:

![](/bitcoincore_offline_signing/verify_core_signatures.png#center)

Un s'ha verificat que les signatures son correctes es pot descomprimir el fitxer, dins de la carpeta que genera interesa el binari `bitcoin-qt`, és a dir el binari de Bitcoin Core amb interfície gràfica.
Aquest binari serà el que s'utilitzara en els dos ordinadors: com a air-gapped wallet i com a watch-only wallet.

![](/bitcoincore_offline_signing/bitcoin_qt_binary_selection.png#center)

S'ha de copiar el binari dins d'un dels USBs completament nous.

> :bulb: *Com a pas opcional es pot copiar també la carpeta amb les claus públiques dels desenvolupadors de Core i els fitxers amb els hashos i les signatures. Això pot ser útil per si en un futur es vol actualitzar el binari, tenir ja importades les claus per poder veríficar-ne l'autenticitat des de l'ordinador air-gapped.*

![](/bitcoincore_offline_signing/usb_bitcoincore_share.png#center)

### Pas 3. Instalar Bitcoin Core a l'ordinador aillat d'Internet i generar una wallet nova amb les claus privades.

Per instalar Bitcoin Core es copia el fitxer `bitcoin-qt` des de l'USB a l'ordinador air-gapped. S'ha de copiar a la carpeta `Persistent` i a continuació executar-lo.
![](/bitcoincore_offline_signing/start_core1.png#center)

Quan s'inicia Bitcoin Core per primer cop et pregunta on vols crear el directory `.bitcoin`. Seleccionarem `custom data directory` i colocarem el directori `.bitcoin` dins la carpeta `Persistent` així la wallet i tots els fitxers necesaris no s'esborraran en reiniciar el sistema.

L'opció de límitar el _block storage_, és a dir quant espai deixem que Bitcoin Core utilitzi per emmagatzemar la blockchain, és indiferent. Com que l'ordinador estara sempre desconectat de la xarxa no arribara mai a descargar-se la blockchain. Per aquest mateix motiu sempre que s'obri Bitcoin Core surtira una pantalla d'alerta on es mostra el procés de sincronització, encallat sempre al `0.00%`. El podeu amagar prement el botó `Hide`.

![](/bitcoincore_offline_signing/start_core2_sync.png#center)

Un cop fet això ja podem crear la nostra wallet. Per fer-ho s'ha de tocar el boto `File`, a dalt a l'esquerra, i en el menú desplegable `Create Wallet`.
Posem un nom a la wallet i seleccionem l'opció `Encrypt Wallet`.

![](/bitcoincore_offline_signing/create_wallet1.png#center)

A continuació demana una `passphrase` per encriptar la wallet. Sense aquesta passphrase no es pot desencriptar el fitxer, així que és important no perdre-la.

> :warning: **ALERTA!**\
> *Qualsevol persona amb el fitxer de la wallet `.dat` i la passphrase podrà accedir als teus fons, és important que la passhprase sigui complicada perque no sigui facilment esbrinable. La passphrase en aquest context és una contrasenya per xifrar el fitxer de la wallet, no te res a veure amb la passphrase que s'utilitza en el [BIP39](https://bips.dev/39)*

![](/bitcoincore_offline_signing/create_wallet2.png#center)

> :warning: **ALERTA!**\
> *És molt important que tant el binari `bitcoin-qt` com el fitxer de la wallet `.dat`, estiguin a la carpeta `Persistent` sinó és així els fitxer s'esborraran automàticament en apagar l'ordinador.*

Un cop la wallet s'ha creat, ens interesa fer-ne copies de seguretat. Per fer-ho anirem altre cop a `File` i a continuació `Backup Wallet`.

<div style="display: flex; gap: 10px; justify-content: center; flex-wrap: wrap;">
  <img src="/bitcoincore_offline_signing/backup_wallet1.png" style="max-width: 45%; min-width: 300px;" />
  <img src="/bitcoincore_offline_signing/backup_wallet2.png" style="max-width: 45%; min-width: 300px;" />
</div>

> :bulb: *Aquestes còpies de seguretat són còpies completes de la wallet. Seria interesant emmagatzemar-les en diferents USBs i/o microSDs i guardar-les en diferents localitzacions. Si et preocupa la seguretat també pots xifrar aquests USBs i microSDs. Qualsevol persona amb accés a ells podrà gastar els fons. Xifrar les còpies de seguretat també és una mesura de privacitat, ja que qualsevol pot llegir la xpub en text clar de qualsevol fitxer `wallet.dat.`*

### Pas 4. Exportar els descriptors públics de la wallet

Com que l'ordinador air-gapped no pot sincronitzar el node de Bitcoin, la wallet no veurà quants fons tenim ni quines transaccions s'han fet.
Per tant s'han d'exportar els descriptors i les claus públiques per importar-les a l'altre ordinador.

Aquest pas és el més senzill i és exactament idèntic al pas anterior on hem creat una còpia de seguretat de la wallet, però en comptes de seleccionar la opció `Backup Wallet` seleccionem l'opció `Export WatchOnly Wallet`

![](/bitcoincore_offline_signing/export_watchonly_wallet1.png#center)

El fitxer `.dat` generat l'emmagatzem en un USB nou.

### Pas 5. Importar els descriptors públics a l'ordinador amb conexió a Internet

Important els descriptors públics a l'ordinador on sí que tenim conexió a Internet podrem veure l'historial de transaccions i els fons disponibles.

Per fer-ho arranquem el mateix binari `Bitcoin-qt` on veurem que allà la sincronització sí que avança.

![](/bitcoincore_offline_signing/sync_bitcoin_core.png#center)

Per importar els descriptors farem click a `File` i `Restore Wallet`.

![](/bitcoincore_offline_signing/restore_wallet1.png#center)

S'obrira un menú per seleccionar el fitxer de la wallet. Seleccionem el fitxer de la `watch-only-wallet` que hem guardat a l'USB i l'obrim.

![](/bitcoincore_offline_signing/restore_wallet2.png#center)

El programa ens demanarà posar un nom i un cop acceptem la wallet ja estara carregada.

![](/bitcoincore_offline_signing/restore_wallet3.png#center)

> :bulb: *Al ser només watch-only aquesta wallet ens permetrà fer tot, menys firmar transaccions, és a dir, podrà crear transaccions, veure transaccions entrants, veure els fons disponibles, portar la comptabilitat, etc.*

Podem comprovar que tot esta correcte generar una nova direcció per rebre fons en els dos ordinadors. Les direccions haurien de coincidir.

<div style="display: flex; gap: 10px; justify-content: center; flex-wrap: wrap;">
  <img src="/bitcoincore_offline_signing/check_address_offline.png" style="max-width: 45%; min-width: 300px;" />
  <img src="/bitcoincore_offline_signing/check_address_online.png" style="max-width: 45%; min-width: 300px;" />
</div>

### Pas 6. Fer pagament amb aquest setup

El procediment per fer pagaments amb aquesta configuració de dos ordinadors és exactament el mateix que si utilitzessim una hardware wallet.

Primer de tot es necesita crear una PSBT (_partially signed bitcoin transaction_). Per fer-ho, des de l'ordinador on hi ha conexió a Internet cliquem el boto de `Send`, posem la direcció on volem enviar els fons, la quantitat, etc.

![](/bitcoincore_offline_signing/create_psbt1.png#center)

Veurem que en comptes d'un boto per fer broadcast, surt `Create Unsiged`. Cliquem allà i sortira una pantalla de confirmació amb els detalls.

![](/bitcoincore_offline_signing/create_psbt2.png#center)

El boto `Send` estarà bloquejat ja que obviament no podem enviar una transacció sense firmes.
Cliquem boto `Create Unsiged` i s'obrira un altre petit menú que ens diu que la PSBT s'ens ha copiat al clipboard i donarà l'opció per guardar-la en un fitxer.

![](/bitcoincore_offline_signing/create_psbt_2_2.png#center)

Aquí tenim dues opcions per transferir la transacció a l'ordinador desconectat d'Internet.

- Opció 1. Guardar la PSBT en un fitxer i transferir el fitxer utilitzant una microSD o un USB completament nou.
- Opció 2. Generar un codi QR a partir de la PSBT per escanejar des de la càmera de l'altre ordinador.

En aquesta guia seguirem l'opció dos. Com que no volem generar el fitxer de la PSBT podem fer click a `Discard`.

> :warning: **ALERTA!**\
> *Si tries fer servir microSDs o USBs assegurat que tots els dispositius són nous de fàbrica! No connectis a l'ordinador air-gapped cap USB o microSD que ha estat prèviament connectat a un dispositiu amb connexió a Internet, podria estar infectat amb malware i intentar filtrar el fitxer de la teva wallet o les teves claus privades.*

Podeu utilitzar qualsevol programa en el que confieu per crear el codi QR, si esteu en linux i us sentiu una mica comodes podem utilitzar la terminal amb la comanda:

```bash
echo PSBT | qr
```

Heu de substituir `PSBT` per la PSBT que Bitcoin Core us ha copiat. Aquesta comanda us retornara un codi QR.

![](/bitcoincore_offline_signing/create_psbt3.png#center)

A continuació des de l'ordinador air-gapped s'ha d'escanejar el QR, altre cop podeu utilitzar qualsevol programa.

> :warning: **ALERTA!**\
> *Tingueu en compte que si heu d'instalar algún programa en concret haurieu d'instalar-lo abans de crear la wallet, per enviar que, en cas d'instalar malware, pugui filtrar les claus privades de la wallet.*

En aquest cas faig servir `zbarcam` des de la terminal. Quan troba un codi QR el llegeix i printa per la terminal el text.

![](/bitcoincore_offline_signing/sign_psbt1.png#center)

Com es pot comprovar el valor que printa és la PSBT que s'ha creat previament.
Per firmar-la obrim la wallet a Bitcoin Core i anem a `File` i després `Load PSBT from clipboard`. En cas de haver utilitzat un USB o una microSD s'ha de fer servir l'opció `Load PSBT from file`.

![](/bitcoincore_offline_signing/sign_psbt2.png#center)

Un cop la PSBT es carrega surtira un menú per firmar la PSBT, es fa click a `Sign Tx` i després a `Copy to Clipboard`. Podeu fer click a `Save` en cas de que vulgueu fer servir un USB o microSD per copiar la transacció a l'altre ordinador.

![](/bitcoincore_offline_signing/sign_psbt3.png#center)

En cas de voler utilitzar QRs podem utilitzar altre cop la comanda:

```bash
echo Tx | qr
```

Aquest cop substituint `Tx` per el valor copiat.

![](/bitcoincore_offline_signing/sign_psbt4.png#center)
![](/bitcoincore_offline_signing/sign_psbt5.png#center)

Des de l'ordinador amb conexió a Internet podem escanejar de la mateix a manera que abans el QR.

![](/bitcoincore_offline_signing/broadcast_psbt1.png#center)

Des de Bitcoin Core altre vegada obrim la PSBT fent click a `File` i després `Load PSBT from clipboard`.

![](/bitcoincore_offline_signing/broadcast_psbt2.png#center)

A continuació es fa click a `Broadcast Tx` i veurem com les lletres blanques sobre un fons verd diuen que la transacció s'ha enviat correctament.

![](/bitcoincore_offline_signing/broadcast_psbt3.png#center)

Amb això conclou la guía, per seguir fent pagaments simplement s'ha de repetir el pas 6.
Per rebre pagaments, des de l'ordinador conectat a Internet es pot clickar a la pestanya receive per generar addresses noves.

La següent secció és una secció extra de la guía que explica com podem fer còpies de seguretat físiques de la wallet.
Si no t'interessa fer còpies físiques, pots parar de llegir aquí.

## Secció Extra - Còpies de seguretat físiques

En els passos anteriors només s'han fet còpies de seguretat digitals, guardant còpies de la wallet en diversos USBs o microSD, però potser volem tenir algunes còpies de seguretat físiques o en paper.

Bitcoin Core no implementa el [BIP39](https://bips.dev/39) per tant no podem fer una còpia de les 12 o 24 paraules i ja. El que hem de fer és guardar el descriptor amb la clau privad (`xprv`).

Per fer-ho obrirem la consola de Bitcoin Core des de l'ordinador air-gapped. I desbloquejarem la wallet amb la comanda

```
walletpassphrase PASSPHRASE 60
```
Substituint `PASSPHRASE` per la contrasenya que s'ha triat en el pas 3.
![](/bitcoincore_offline_signing/backup_descriptors1.png#center)

A continuació llistem els descriptors amb 

```
listdescriptors true
```

El valor `true` indica que es llistin els descriptors privats.

![](/bitcoincore_offline_signing/backup_descriptors2.png#center)

Això retorna un llistat amb els descriptors. Hem de buscar en algún d'ells el valor que comença per `xprv` i copiar-lo.

> :bulb: *A la imatge es veu que comença amb `tprv` perquè s'ha utilitzat testnet*.

![](/bitcoincore_offline_signing/backup_descriptors3.png#center)

Aquest valor es pot apuntar en un paper i serà suficient per recuperar la wallet.

> :bulb: *En cas que s'hagin creat els descriptors a mà utilitzant [derivation path](bips.dev/32) personalitzats, s'hauria de copiar el descriptor sencer amb el derivation path. En aquest cas, com que hem utilitzat el per defecte, podem confiar en que Bitcoin Core mantindra els derivation path estàndards i no cal apuntar-se'ls.*

Una altre manera de fer la còpia de seguretat és guardar la `xprv` en un codi QR. Per fer-ho podem utilitar les mateixes eines que en el pas 6 per codificar la PSBT en un QR.

![](/bitcoincore_offline_signing/backup_descriptors5.png#center)

Imprimir el QR no és bona idea ja que no volem que cap altre dispositiu la vegi mai, una opció és posar un paper sobre la pantalla i calcar el codi QR.
Un cop el tens en paper, pots provar d'escanejar-lo des del dispositiu air-gapped per veure si el pot llegir correctament.
Personalment, recomanaria fer múltiples còpies de seguretat en formats diferents, mentre que un QR és una bona opció, copiar a mà la clau privada (xprv) és un bon complement.
