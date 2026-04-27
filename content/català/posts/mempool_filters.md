+++
title = 'Una visió tècnica i opinió sobre els filtres de la mempool'
date = 2026-04-27
draft = false
tags = ["Nodes", "Mempool", "P2P", "Mining", "Policy"]
categories = ["Opinion"]
+++


Traducció feta per Trid^^!!

# Què és la mempool?

La definició més senzilla de la mempool és: ***la mempool*** *és una cache que emmagatzema transaccions vàlides per consens, però encara no confirmades*.

## **Però per què necessitem una mempool?**

La mempool és un recurs important per a cada node i permet principalment que la xarxa peer-to-peer (P2P) funcioni.

Cada \~10 minuts, quan es troba un bloc nou, els nodes sense mempool experimenten un pic en el consum d'amplada de banda, és a dir, els nodes reben de cop moltes dades, a més d'entrar en un període computacionalment intens validant cada transacció del bloc. En canvi, els nodes amb mempool ja han vist totes o la majoria de les transaccions dins d'un bloc nou i les tenen emmagatzemades a la seva mempool.

Amb [Compact block relay](https://bitcoinops.org/en/topics/compact-block-relay/) ([BIP-152](https://github.com/bitcoin/bips/blob/master/bip-0152.mediawiki)), els nodes només descarreguen la capçalera del bloc anterior més algunes pistes sobre quines transaccions hi ha dins del bloc. Després, el node reconstrueix el bloc recuperant les transaccions de la mempool i demana únicament les que falten al node que ha enviat la capçalera del bloc anterior. Això és molt més eficient i ràpid que sol·licitar el bloc sencer, perquè cal enviar menys dades, i la validació de les transaccions ja s'ha fet prèviament. Utilitzar una mempool i compact blocks augmenta dràsticament la velocitat de propagació de blocs a tota la xarxa i, per tant, redueix el risc de "stale blocks" (blocs obsolets).

La mempool també és útil per estimar les comissions (fees) que has de pagar perquè una transacció sigui minada en el rang de temps que vulguis. L'espai de bloc es ven en un mercat de subhastes basat en comissions. Observant les transaccions pendents a la mempool, pots estimar quina és la comissió mínima que els altres estan disposats a pagar per l'espai de bloc i quines ofertes han tingut èxit en el passat.

Per a més informació sobre per què tenim una mempool, llegeix: [Waiting for confirmation #1: why do we have a mempool?](https://bitcoinops.org/en/newsletters/2023/05/17/#waiting-for-confirmation-1-why-do-we-have-a-mempool)

## Regles de consens (Consensus Rules) vs Filtres (Policy Rules)

Quan parlem de filtres, ens referim a les regles que decideixen quines transaccions són estàndard. Si una transacció és estàndard, entra a la nostra mempool i es retransmet per tota la xarxa. No afegim transaccions a la nostra mempool ni retransmetem transaccions que considerem no estàndard.

Fixeu-vos que en la definició anterior no parlem de blocs ni de la blockchain. Això és perquè aquests filtres no s'apliquen a les transaccions que ja són dins d'un bloc. Si aquests filtres s'apliquessin a les transaccions dels blocs, tindríem múltiples bifurcacions de la cadena a causa de discrepàncies entre nodes.

Un exemple senzill: Suposem que els filtres de la mempool també s'apliquen als blocs. L'Alice i en Bob tenen filtres diferents: l'Alice només accepta transaccions que moguin com a mínim 1 BTC, mentre que en Bob no té cap quantitat mínima. En algun moment, quan un miner troba un bloc amb una transacció de menys d'1 BTC, l'Alice descartaria aquest bloc, però en Bob l'acceptaria. Al final, estarien en dues versions diferents de la blockchain!

Les regles que s'apliquen a les transaccions dins dels blocs s'anomenen regles de consens (consensus rules) i aquestes regles han de ser aplicades de manera compatible per tots els nodes. Si no és així, és probable que la blockchain es bifurqui (es divideixi). Alguns exemples d'aquestes regles són: una transacció no pot tenir sortides que gastin més diners del que té a les entrades, les transaccions no poden gastar dues vegades les mateixes monedes (double spend), entre d'altres.

## **Llavors, quin és el paper dels filtres (policy rules)?**

El teu node és un ordinador connectat a una xarxa peer-to-peer amb moltes altres entitats desconegudes que poden llançar un atac de denegació de servei (DoS - denial of service) creant missatges que provoquen que el node es quedi sense memòria i es bloquegi, o que gasti els seus recursos computacionals i ample de banda en dades sense sentit en lloc d'acceptar nous blocs.

Com que aquestes entitats són desconegudes per disseny, el teu node no pot saber si s'està connectant a un company honest o maliciós, i no pot bloquejar-los efectivament per evitar ser atacat. Per això, és imprescindible implementar filtres (polítiques) contra el DoS per limitar el cost de mantenir un node complet (full node).

> ***Un exemple real molt senzill:*** Un atacant podria crear una transacció amb moltes signatures, de les quals totes menys l'última són vàlides. Això faria que el teu node gastés molts recursos i temps validant la transacció fins que fallés just al final. Això es prevé amb un filtre que imposa un límit en el nombre d'operacions de signature que pot tenir una transacció. Fixeu-vos que, com que aquesta regla només s'aplica a les transaccions que reps fora dels blocs, si una transacció amb moltes signatures acaba dins d'un bloc vàlid, el teu node la validarà i l'emmagatzemarà.

Per a una explicació més detallada sobre què són els filtres (policy rules) i per què s'utilitzen, consulteu [Waiting for confirmation #5: Policy for Protection of Node Resources](https://bitcoinops.org/en/newsletters/2023/06/14/#waiting-for-confirmation-5-policy-for-protection-of-node-resources).

### OP\_RETURN

L'OP\_RETURN és un codi d'operació (opcode) del llenguatge de scripting que Bitcoin utilitza per incloure dades arbitràries en una transacció. Aquest opcode marca la sortida de la transacció com a "no gastable", assegurant que aquesta sortida no es pugui utilitzar com a entrada en una transacció posterior.

El pes màxim de les dades que es poden incrustar utilitzant OP\_RETURN està limitat principalment per la mida del bloc (una regla de consens). Històricament, hi ha hagut un filtre (policy rule) que limitava la quantitat de dades que es podien afegir. El darrer filtre limitava les sortides OP\_RETURN a 83 bytes. Aquest límit està augmentat a Bitcoin Core, per defecte, a 100.000 bytes, ja que hi ha un filtre per al pes màxim de la transacció. Recordeu que un filtre només afecta les transaccions que el node retransmet; no fa que una transacció sigui invàlida si es veu dins d'un bloc.

#### **¿Permetre un OP\_RETURN més gran incentiva el correu brossa (spam)?**

No veig per què ho faria. És cert que és probable que ara veiem més OP\_RETURN amb quantitats més grans de dades. També és dubtós que eliminar el límit de l'OP\_RETURN augmenti la quantitat total de dades a la blockchain, ja que és més car utilitzar un OP\_RETURN que utilitzar altres protocols com els [ordinals](https://docs.ordinals.com/inscriptions.html). Com assenyala Vojtěch al [Bitcoin stack exchange](https://bitcoin.stackexchange.com/a/122322/145161), l'OP\_RETURN deixa de ser econòmicament interessant si les dades a inscriure superen els 143 bytes.

#### **¿Un límit d'OP\_RETURN més gran provocarà un creixement del conjunt UTXO?**

Per contextualitzar, el conjunt de les UTXOs (UTXO set) és una base de dades on els nodes emmagatzemen les sortides de transacció no gastades (unspent transaction outputs), és a dir, totes les monedes disponibles per ser gastades. La mida del conjunt UTXO és important per raons de rendiment i descentralització: com més gran sigui el conjunt UTXO, més grans són els recursos necessaris per executar un node.

Com que l'OP\_RETURN fa que una sortida no es pugui gastar, no cal que s'emmagatzemi al UTXO set, per tant no el fa més gran. En altres paraules, és la manera més neta i saludable d'incrustar dades a la blockchain.

#### **¿Per què és millor que altres mètodes anti-brossa (spam)?**

Fixeu-vos que quan diem "millor" volem dir "la manera més saludable". La primera raó ja s'ha esmentat al paràgraf anterior: crear sortides OP\_RETURN no infla el conjunt UTXO, però hi ha altres punts positius.

Un altre punt important és que mostra les dades directament a la sortida. Altres mètodes incrusten les dades utilitzant SegWit o scripts de Taproot. Aquests scripts són "secrets" fins que la sortida es gasta; un cop es gasta, les dades es revelen a les entrades de la següent transacció. Això significa que per incrustar dades necessiten crear dues transaccions en lloc d'una sola. Aquesta manera de funcionar pot incentivar la creació sortides falses (dummy outputs).

Un punt addicional a considerar és que els scripts OP\_RETURN no disposen del descompte de segwit (witness discount).

### Filtres

Com he esmentat anteriorment, el paper principal dels filtres (policy rules) és evitar que els nodes pateixin atacs de denegació de servei (DoS), però no evitar que les transaccions arribin als blocs.

#### **¿Com ens podem saltar els filtres?**

Mentre hi hagi un incentiu econòmic per fer-ho, saltar-se aquests filtres és fàcil i es pot fer de diverses maneres:

- Enviar directament a l'API d'un pool de mineria: Els pools de mineria de vegades tenen l'opció de rebre transaccions directament sense que passin per la xarxa peer-to-peer. Això permet al remitent evitar tots els filtres i entrar en un bloc amb certa seguretat.

- Utilitzar un node permissiu amb connexions preferents: Nodes amb filtres menys estrictes (com [Libre Relay](https://geyser.fund/project/librerelay)) reenviaran aquestes transaccions filtrades i, després de ser reenviades diverses vegades, acaben arribant a la mempool d'algún miner. De vegades fins i tot implementen algún tipus de connexió preferent per assegurar que hi hagi una ruta entre aquests nodes i els miners. D'aquesta manera, les transaccions arriben als miners encara que un gran percentatge de nodes de la xarxa les filtri.


#### **¿Els filtres funcionen?**

Un dels problemes principals en aquest debat és el desacord sobre la definició de "funcionar". D'una banda, hi ha arguments que diuen que els filtres funcionen perquè dificulten la propagació de les transaccions, forçant-les a passar per camins menys còmodes, com els que s'han esmentat a la secció anterior. De l'altra banda, hi ha arguments que diuen que els filtres no funcionen perquè no poden evitar que les transaccions acabin dins d'un bloc.

Al meu parer, els filtres funcionen, però funcionen per a la finalitat per a la qual van ser creats: evitar que els nodes pateixin atacs de denegació de servei (DoS). La seva funció no és evitar que les transaccions arribin als blocs. Per tant, si el que volem és que no s'incrustin dades a la blockchain, hauríem de buscar una solució que abordi aquest problema específic.

#### **¿Per què els filtres no són una solució i poden ser fins i tot perjudicials?**

Fem una mica de resum del que ja hem parlat per poder respondre a aquesta pregunta amb tot el context necessari. Tenim una mempool als nostres nodes, que és una memòria (cache) de transaccions no confirmades que pensem que poden acabar en els propers blocs. Com que necessitem que els blocs es propaguin ràpidament per evitar blocs obsolets, tenim un protocol anomenat Compact Blocks. Aquest protocol ens permet reconstruir i difondre els nous blocs més ràpidament perquè ja tenim les transaccions del bloc a la nostra mempool.

Com que Bitcoin és una xarxa sense permisos (permissionless), no podem saber si els nostres companys (peers) són maliciosos fins que ja s'han connectat a nosaltres. Això significa que necessitem tenir mesures de protecció per evitar patir atacs de denegació de servei (DoS). Ens protegim a nosaltres mateixos filtrant el que acceptem i transmetem a la resta de la xarxa. Aquests filtres no eviten que les transaccions arribin als blocs, només fan que la propagació de les transaccions sigui més difícil, però no impossible.


**Llavors, ¿per què els filtres són una solució perjudicial?** **Perquè** **empenyen cap a la centralització de la mineria!**
Hi ha dues raons per les quals els filtres empenyen cap a la centralització de la mineria, però abans d'entrar-hi, definim què és la centralització de la mineria.

En aquest cas, quan parlem que la mineria estaria centralitzada, ens referim a qui crea les plantilles de bloc (block templates) i no a qui les mina (hashers). Aquí podem identificar dos actors diferents: el creador de la plantilla de bloc, és a dir, qui decideix quines transaccions entren als blocs, i els hashers, tots aquells usuaris amb ASICs, bitaxes, etc. que fan mineria a sobre d'algun pool que fa la feina de creació de la plantilla.

Hi ha dues raons diferents per les quals els filtres empenyen cap a la centralització de la mineria.

- Si el teu node filtra transaccions, el protocol compact block no podrà reconstruir el bloc ràpidament perquè hi faltarà informació. Això comportarà una propagació més lenta dels blocs i augmentarà la possibilitat de blocs obsolets (stale blocks). En aquest escenari, els miners més petits són els més perjudicats, perquè els miners grans probablement estan millor connectats entre ells. Això fa que probablement la major part del hashrate de la xarxa mini a sobre del bloc del miner més gran, descartant així tota la feina feta pel miner petit.

- Si un usuari tria enviar una transacció fora de banda (no utilitzar la xarxa P2P normal i saltant-se els nodes intermedis), en lloc de fer-la circular per la xarxa peer-to-peer utilitzant, per exemple, una API pública d'un pool de mineria, probablement triarà un pool gran. Això és perquè enviar-la a un pool gran augmenta les possibilitats que la transacció sigui minada més ràpidament. Aquestes transaccions fora de banda normalment van acompanyades d'una comissió addicional. Aquesta comissió addicional la podríem considerar [MEVil](https://bluematt.bitcoin.ninja/2024/04/16/stop-calling-it-mev/). Resumint, MEVil causa centralització de la mineria perquè els blocs d'aquests pools grans, on els usuaris envien transaccions fora de banda, guanyen més diners en comissions que els altres. Això empeny als hashers a apuntar el seu hashrate cap a aquests pools per maximitzar beneficis.

Es pot veure un exemple de filtres que no funcionen amb el cas (segona meitat del 2025) de les transaccions per sota d'1 sat/vB. TLDR: Un gran grup de miners va començar a [acceptar transaccions que pagaven menys d'1 sat/vB](https://x.com/mononautical/status/1947530080475091159) mentre que els filtres dels nodes rebutjaven aquestes transaccions. Quin va ser el resultat? Aquestes transaccions van ser minades igualment (els filtres no van funcionar) i el compact block es va trencar, [la propagació de blocs es va endarrerir significativament](https://github.com/bitcoin/bitcoin/pull/33106#issuecomment-3155627414).

En resum, no només els filtres no poden evitar que l'spam arribi a la blockchain, sinó que a més són perjudicials per a la descentralització de la mineria. No són una solució per a aquest problema; és com disparar-nos al nostre propi peu.

## ¿Segwit i Taproot van causar això?
No, segur que no. OP\_RETURN ja existia abans de SegWit i Taproot i es podia utilitzar per incrustar dades a la blockchain. I els altres mètodes que s'estan utilitzant podien fer servir P2SH.

Tot i que SegWit i Taproot no en són la causa, no descartaria que contribueixin indirectament al creixement de l'spam. SegWit va venir amb un descompte (witness discount) que va fer més barat incrustar dades, i Taproot va augmentar la quantitat de dades que es poden adjuntar a una entrada, ja que no té el límit de 10.000 bytes que té P2WSH.

## ¿Són Datum i Sv2 la solució?

Datum i Sv2 són dues millores importants per descentralitzar la mineria, ja que permeten als usuaris crear les seves pròpies plantilles de bloc i continuar compartint la recompensa dels blocs minats. El problema és que només resolen la pregunta "Com descentralitzem la creació de plantilles de bloc?". La resposta a "com ho fem" no importa si tenim forces de centralització que empenyen cap als pools de mineria centralitzats.

Els miners (hashers, és a dir, persones amb dispositius de mineria com ASICs, BitAxes, etc.) normalment estan incentivats econòmicament. Al cap i a la fi, la mineria és un negoci i l'objectiu és extreure el màxim valor possible amb el mínim cost, alhora que es maximitzen els beneficis. Si hi ha un pool gran que mina blocs amb una recompensa addicional gràcies a les transaccions fora de banda (MEVil), els miners tenen un fort incentiu per apuntar la seva potència de mineria cap a aquest pool. Això fa que Datum i Sv2 siguin inútils i irrellevants.

Un altre problema amb Datum, Sv2 i els filtres és la teoria de jocs darrere de col·laborar amb desconeguts. Idealment, un miner voldrà unir-se a altres miners que també treballin per maximitzar beneficis. ¿Per què un miner que maximitza beneficis col·laboraria amb algú que exclou algunes transaccions i, per tant, no obté totes les recompenses possibles? Per col·laborar, les persones han d'estar impulsades pels mateixos objectius i incentius. Confiar que els miners faran la mateixa feina per menys diners mentre altres recullen aquests diners només per ideals no és realista i deixa el resultat del problema de la centralització de la mineria en mans de desconeguts.

En conclusió, crec realment que Datum i Sv2 són excel·lents solucions i millores importants, però el seu objectiu final és descentralitzar la mineria per evitar que els grans pools censurin transaccions. No són una solució contra la centralització de la mineria impulsada per incentius econòmics.



