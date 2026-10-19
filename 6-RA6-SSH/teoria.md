# Serveis de xarxa — Accés remot i SSH

## Objectius didàctics

En acabar aquesta unitat, has de ser capaç de:

1. Distingir **confidencialitat, integritat i autenticació**: què protegeix cadascuna i per què són necessàries.
2. Diferenciar el **xifratge simètric** de la **criptografia asimètrica**, identificant les claus que utilitzen.
3. Identificar la funció de **SHA-2, AES i RSA**, sense confondre un resum criptogràfic amb un xifratge.
4. Distingir **xifrar, calcular un resum i signar**, explicant què aconsegueix cada operació.
5. Explicar la funció de **Diffie-Hellman**: obtenir un secret compartit sense enviar-lo, i comprendre per què també cal autenticar l'intercanvi.
6. Diferenciar les **claus d'identitat del servidor i de l'usuari** de les **claus simètriques de sessió**.
7. Distingir l'**autenticació del servidor** de l'**autenticació de l'usuari**, amb contrasenya o clau pública.
8. Explicar què són una **empremta** i el fitxer **`known_hosts`**, i com actuar davant una clau desconeguda o un canvi de clau.
9. Explicar la funció bàsica d'un **certificat digital** i distingir la confiança mitjançant una **CA** de la confiança directa en una clau.
10. Diferenciar **SSH i TLS**, i entendre que **HTTPS és HTTP protegit amb TLS**.
11. Identificar **entre quins extrems queda xifrada la comunicació**, tant en una connexió SSH com en un túnel.

## Per què necessitem accés remot

Quan un servidor està en un rack, en un CPD o simplement en una màquina virtual d'un laboratori, no ens és pràctic anar-hi físicament cada vegada que vulguem administrar-lo. L'**accés remot** permet controlar un ordinador des d'un altre a través de la xarxa, com si estiguéssim asseguts davant seu.

Hi ha dos grans blocs d'eines d'accés remot:

- **Accés en mode text (terminal remot)** — l'administrador rep una consola de comandes del servidor. És lleuger, ràpid i permet fer moltes tasques d'administració. L'estàndard actual és **SSH**.
- **Accés en mode gràfic (escriptori remot)** — l'administrador rep el mateix escriptori que veuria en pantalla. Necessari per clients Windows o per usuaris sense pràctica en línia de comandes. Els protocols habituals són **VNC** i **RDP**.

### Protocols insegurs i protocols segurs

Els protocols antics d'accés remot (`telnet`, `rlogin`, `rsh`, `ftp`) transmetien tot el trànsit —incloent contrasenyes— **en clar** per la xarxa. Qualsevol amb accés al mateix segment de xarxa podia llegir-ho amb un capturador de paquets com Wireshark. Avui dia estan pràcticament abandonats.

Els substituts moderns (**SSH, SFTP, HTTPS, IMAPS, SMTPS**...) protegeixen el trànsit entre el client i el servidor. En SSH distingirem tres funcions: comprovar la identitat del servidor, acordar un secret compartit i xifrar les dades amb claus simètriques.

La **confidencialitat** evita que tercers llegeixin les dades; la **integritat** permet detectar-ne modificacions, i l'**autenticació** comprova la identitat de l'altra part.

El xifratge de la connexió protegeix les dades durant el trajecte entre els seus extrems; el servidor que les rep les desxifra per poder processar-les.

## SHA-2, AES i RSA: resumir, xifrar i signar

Abans d'explicar SSH, cal distingir tres eines criptogràfiques que també apareixeran en HTTPS i correu electrònic. **SHA-2, AES i RSA no són tres tipus equivalents de xifratge**: fan funcions diferents.

### SHA-2: calcular un resum de les dades

**SHA-2** és una família de funcions de resum criptogràfic, o *hash*. En formen part **SHA-256** i **SHA-512**. Una funció de resum transforma unes dades, siguin curtes o llargues, en un valor de mida fixa. Per exemple, SHA-256 produeix 256 bits, que habitualment es mostren com 64 caràcters hexadecimals.

- Les mateixes dades produeixen sempre el mateix resum.
- Canviar una lletra, un espai o una majúscula produeix, amb una probabilitat molt alta, un resum diferent.
- No utilitza una clau secreta i **no xifra les dades**. No hi ha una operació de desxifratge que permeti recuperar el text original a partir del resum. Tanmateix, es poden provar textos candidats i comparar-ne els resums.

Un resum permet comprovar si unes dades coincideixen amb les esperades, sempre que el resum de referència sigui fiable. **No demostra, tot sol, qui ha enviat les dades**: un atacant que les canviï també podria calcular-ne un resum nou. En SSH veurem resums utilitzats per calcular empremtes de claus i com a part de les signatures.

### AES: xifrar dades amb una clau simètrica

**AES** és un algoritme de xifratge **simètric**: qui xifra i qui desxifra utilitzen la mateixa clau secreta. Les variants AES-128, AES-192 i AES-256 indiquen la mida de la clau en bits.

```text
Text original + clau secreta → xifratge AES → dades xifrades
Dades xifrades + mateixa clau secreta → desxifratge AES → text original
```

AES és ràpid i adequat per protegir moltes dades. Pot xifrar el trànsit d'una sessió SSH o TLS. La dificultat inicial és que tots dos extrems han de disposar de la clau secreta: **AES no resol com l'acorden**. Més endavant veurem com Diffie-Hellman permet obtenir un secret del qual es calculen les claus de sessió.

AES s'utilitza amb un **mode d'operació**, que defineix com es processen les dades. Alguns modes, com GCM, també permeten detectar modificacions; no hem de donar per fet que qualsevol ús d'AES ja protegeix la integritat.

### RSA: utilitzar una clau pública i una de privada

**RSA** és un algoritme de criptografia **asimètrica**. Utilitza un parell de claus relacionades matemàticament:

- **Clau pública**: es pot compartir.
- **Clau privada**: es conserva en secret. Amb mides i paràmetres adequats, obtenir-la a partir de la pública no és viable de manera pràctica.

RSA permet dues operacions diferents, amb procediments específics per a cadascuna:

| Operació | Quina clau s'utilitza primer? | Què fa l'altra part? | Què aconseguim? |
|---|---|---|---|
| Xifrar per a un destinatari | La pública del destinatari xifra | La privada del destinatari desxifra | Protegir la confidencialitat |
| Signar unes dades | La privada de l'emissor signa | La pública de l'emissor verifica | Comprovar la signatura i detectar canvis en les dades signades |

**Signar no és xifrar amb la clau privada**: una signatura no oculta el missatge. A més, una signatura vàlida només es pot atribuir a una identitat si sabem a qui pertany la clau pública utilitzada per verificar-la.

En SSH, si s'utilitzen claus RSA, serveixen per a **signatures d'autenticació**, no per xifrar tot el trànsit. RSA no és l'única opció per signar: també veurem claus ED25519. No cal confondre la mida d'una clau RSA amb la d'una clau AES: són algoritmes diferents i els bits no es comparen directament.

### Ho podem provar amb CyberChef

[CyberChef](https://gchq.github.io/CyberChef/) és una eina que permet combinar operacions al navegador. Escrivim dades a **Input**, arrosseguem operacions a **Recipe** i observem el resultat a **Output**. Per a aquestes proves utilitzarem textos i claus de laboratori.

| Què volem observar? | Prova amb CyberChef | Què hem de comprovar? |
|---|---|---|
| El resum SHA-2 | Busca `SHA2`, selecciona 256 bits i escriu `Hola`. Després canvia-ho per `hola`. | El resum canvia i manté la mateixa longitud. No hi ha cap clau ni operació de desxifratge. |
| El xifratge simètric | Xifra un text amb `AES Encrypt` i recupera'l amb `AES Decrypt`. | Cal utilitzar la mateixa clau i els paràmetres corresponents en totes dues operacions. |
| Les claus asimètriques | Crea un parell de claus de prova amb `Generate RSA Key Pair`, seleccionant 2048 bits i format PEM. Utilitza la pública amb `RSA Encrypt` i la privada amb `RSA Decrypt`, amb el mateix esquema de xifratge. | La clau pública permet xifrar un text curt, però cal la privada per recuperar-lo. |

Per a una primera prova d'AES, podem seleccionar **CBC**, posar la clau `0123456789abcdef` en format **UTF8** i el vector d'inicialització **IV** `abcdef0123456789`, també en **UTF8**. Són 16 bytes cadascun. A `AES Encrypt`, seleccionem entrada `Raw` i sortida `Hex`; a `AES Decrypt`, entrada `Hex` i sortida `Raw`, conservant clau, IV i mode. L'IV és un valor inicial que necessita aquest mode; no és una segona clau secreta. **Aquests valors fixos només serveixen per seguir l'exemple**: en un ús real cal generar i gestionar les claus i els IV segons els requisits del mode.

L'objectiu de la prova és distingir les funcions, no reproduir una connexió SSH completa: **SHA-2 resumeix; AES xifra amb una clau compartida; RSA utilitza un parell de claus i pot servir per xifrar o signar, segons l'operació.**

## SSH: el model de confiança

SSH (*Secure Shell*) és el protocol estàndard per administrar servidors Linux/Unix des de fa dècades. Escolta al port **22/TCP** per defecte.

Per seguir els exemples, imaginem que la Laia vol accedir al compte `alumne` d'un servidor:

- **Usuari**: la persona que es connecta, en aquest cas la Laia.
- **Client SSH**: el programa `ssh` que executa al seu ordinador.
- **Servidor SSH**: el programa `sshd` de l'ordinador remot, que rep la connexió i gestiona l'accés.
- **Compte remot**: el compte del servidor al qual vol accedir, en aquest cas `alumne`. Pot tenir un nom i una contrasenya diferents dels del seu ordinador.

En aquestes pràctiques utilitzarem el model de **confiança directa** amb `known_hosts`. OpenSSH també admet certificats propis i autoritats de certificació, però són diferents dels certificats X.509 d'HTTPS i queden fora d'aquesta unitat.

Al llarg de la connexió apareixeran claus amb funcions diferents:

| Claus | Quan es preparen? | Per a què serveixen? |
|---|---|---|
| Claus de host del servidor | Habitualment en instal·lar o preparar el servei SSH | Autenticar el servidor |
| Claus d'autenticació de l'usuari | Quan es configura l'accés amb `ssh-keygen` | Autenticar l'usuari com a alternativa a la contrasenya |
| Claus simètriques de sessió | Automàticament durant la connexió | Xifrar el trànsit |

**Podem connectar-nos amb contrasenya sense generar claus d'usuari.** El client i el servidor preparen automàticament les claus de sessió necessàries per xifrar la comunicació.

### Xifrar ≠ tenir un certificat

Aquest és un dels malentesos més freqüents:

- **Xifrar** és una **acció**: transformar dades llegibles en il·legibles amb una clau.
- Un **certificat** és un document digital signat per una autoritat que vincula una **clau pública** amb una identitat (un nom de servidor, en el cas de TLS).

El certificat en si no xifra res: vincula una clau pública amb una identitat. En una connexió moderna amb certificat, el servidor també ha de demostrar que disposa de la privada corresponent mitjançant una signatura. L'intercanvi de claus permet obtenir el secret del qual es deriven les claus simètriques que xifren el trànsit.

**En el model d'SSH que utilitzarem, confiem directament en la clau del servidor**: la primera vegada que ens hi connectem, si l'acceptem, queda guardada al fitxer `~/.ssh/known_hosts` del nostre ordinador. A partir d'aquell moment, cada connexió comprova que la clau presentada correspon a la que tenim guardada.

### Identitat del servidor SSH

El servidor SSH disposa de claus de host, generades habitualment en instal·lar o preparar `openssh-server`. Pot tenir diversos parells de claus, de tipus diferents:

- La **clau privada de host** roman al servidor, dins `/etc/ssh/` (`ssh_host_rsa_key`, `ssh_host_ed25519_key`...).
- La **clau pública de host** s'envia al client en cada connexió.

El client associa la clau de host amb el nom o l'adreça del servidor al qual es connecta. En connexions posteriors, comprova que la clau presentada correspon a la que té guardada per a aquella destinació.

### La primera connexió

En la primera connexió a un servidor SSH que no coneixem, veurem un missatge així:

```
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:xyz123abc...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

El client ens està dient: *no puc verificar automàticament aquesta clau; decideix tu si te'n refies*. Si contestem `yes`, la clau queda guardada a `~/.ssh/known_hosts` i totes les connexions posteriors la compararan amb l'emmagatzemada.

Abans d'acceptar la clau per primera vegada, cal comprovar-ne l'empremta per un canal fiable, per exemple amb l'administrador del servidor. Acceptar `yes` sense aquesta comprovació desa la clau presentada, però no garanteix que sigui la del servidor autèntic.

Una **empremta** és un resum de la clau pública, més curt i fàcil de comparar. No és una altra clau ni és secreta.

Al laboratori, podem consultar l'empremta de la clau ED25519 obrint directament la consola del servidor a VirtualBox i executant:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Si el client mostra una clau ED25519, el valor `SHA256:...` ha de coincidir amb el que acabem de consultar. Si mostra un altre tipus de clau, cal consultar la pública corresponent.

Ho fem des de la consola de VirtualBox perquè encara no hem comprovat que la connexió SSH ens porti al servidor correcte.

El fitxer `known_hosts` conserva les claus de servidor en les quals confiem; no és una autoritat de certificació ni emet certificats.

### Connexions posteriors i atac Man-in-the-Middle (MITM)

En les connexions següents al mateix servidor:

1. El servidor envia la seva clau pública de host.
2. El client la compara amb la guardada a `known_hosts`.
3. Si coincideix, el client també comprova la signatura de l'intercanvi amb aquesta clau pública. Si la signatura és vàlida, queda autenticat el servidor. **El procés de signatura i verificació s’explica més endavant, a l’apartat «Com s’autentica l’intercanvi: la signatura del servidor».**
4. Si la clau **no** coincideix o la signatura no és vàlida, el client no ha de donar el servidor per autèntic.

Un canvi inesperat de clau **s'ha d'investigar**. Pot tenir una causa legítima, com una reinstal·lació, una renovació de claus o un canvi de màquina darrere d'una IP. També pot indicar que un atacant intenta fer-se passar pel servidor: un atac **Man-in-the-Middle (MITM)**.

**No esborrem l'entrada antiga només per fer desaparèixer l'avís.** Primer comprovem la causa del canvi i verifiquem la nova empremta per un canal fiable. Si el canvi és legítim, podem retirar l'entrada antiga amb `ssh-keygen -R IP_SERVIDOR` i acceptar la nova clau verificada.

El missatge d'advertència en aquest cas és molt visible:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
```

Ignorar aquest missatge és una de les errades més greus que es poden fer.

## Xifratge de la sessió: Diffie-Hellman

Per xifrar la sessió, client i servidor necessiten un secret compartit del qual derivar les claus simètriques. Aquí entra en joc l'algoritme **Diffie-Hellman (DH)**. La identitat del servidor es verifica durant aquest intercanvi mitjançant una signatura digital.

### La idea fonamental

Diffie-Hellman permet que **dues parts calculin el mateix secret compartit sense enviar-lo per la xarxa**. Amb paràmetres adequats, un observador no el pot deduir de manera pràctica a partir dels valors públics.

Cada part combina un secret privat propi amb un valor públic rebut de l'altra. Les operacions estan construïdes perquè totes dues arribin al mateix resultat.

### Pas a pas: exemple de DH clàssic

Les fórmules següents mostren una versió clàssica de Diffie-Hellman. SSH també pot negociar altres variants, com ECDH, que compleixen la mateixa funció amb unes altres operacions.

`mod` significa **el residu d'una divisió**. Per exemple, `17 mod 5 = 2`, perquè dividir 17 entre 5 deixa un residu de 2.

1. **Acord públic inicial**: client i servidor es posen d'acord en dos valors que qualsevol pot veure:
   - un número primer molt gran `p`
   - un generador `g` (típicament 2 o 5)

2. **Secrets privats**: cada part genera un número aleatori que **no comparteix mai**:
   - el client tria `a`
   - el servidor tria `b`

3. **Càlculs públics** (aquests sí que viatgen per la xarxa):
   - el client calcula i envia: `A = g^a mod p`
   - el servidor calcula i envia: `B = g^b mod p`

4. **Càlcul del secret compartit** (cadascú fa el seu i coincideixen):
   - el client calcula: `S = B^a mod p`
   - el servidor calcula: `S = A^b mod p`

Els dos valors de `S` són **idèntics**, i el valor `S` **no s'ha transmès mai per la xarxa**.

Coincideixen perquè tots dos càlculs equivalen a `g^(a·b) mod p`.

**Exemple amb nombres petits**, que podem seguir amb una calculadora:

| Pas | Client | Servidor |
|---|---|---|
| Valors públics inicials | `p = 23`, `g = 5` | `p = 23`, `g = 5` |
| Secret privat | `a = 6` | `b = 15` |
| Valor públic calculat i enviat | `A = 5⁶ mod 23 = 8` | `B = 5¹⁵ mod 23 = 19` |
| Secret compartit calculat | `S = 19⁶ mod 23 = 2` | `S = 8¹⁵ mod 23 = 2` |

Tots dos obtenen `2` sense haver-lo enviat. Aquests nombres petits serveixen per entendre el mecanisme, però **no serien segurs en una connexió real**.

Els secrets `a` i `b` són temporals i els genera automàticament cada programa per a l'intercanvi. **No són les claus privades de host ni les claus d'autenticació de l'usuari.**

### Per què és segur

Un observador veu `p`, `g`, `A` i `B`, però no veu `a`, `b` ni `S`. Amb nombres petits pot provar possibilitats fins a trobar els secrets. En DH clàssic, amb grups adequats i nombres prou grans, obtenir el secret a partir dels valors públics és computacionalment inviable amb els mètodes clàssics coneguts.

Trobar `a` a partir de `A = g^a mod p` és el que s'anomena **problema del logaritme discret**. No cal saber resoldre'l: cal entendre que calcular el valor públic és fàcil, però recuperar-ne el secret és molt difícil amb paràmetres adequats.

### Com s'autentica l'intercanvi: la signatura del servidor

Diffie-Hellman, per si sol, protegeix contra un atacant que escolta, però no autentica les parts. Un atacant actiu podria interposar-se i negociar un secret amb el client i un altre amb el servidor.

Per evitar-ho, SSH vincula l'intercanvi a la identitat del servidor:

Un **resum criptogràfic**, o *hash*, és un valor calculat a partir d'unes dades. Si les dades canvien, el resum també canvia amb una probabilitat molt alta.

Una **signatura digital** es calcula amb una clau privada sobre unes dades i es verifica amb la pública corresponent. Permet comprovar que les dades estan vinculades a qui disposa de la privada i que no han estat modificades. **Signar no és xifrar: la signatura no oculta les dades.**

1. El servidor calcula un resum a partir de les dades de negociació, la clau pública de host, els valors públics de DH i el secret compartit. El signa amb la seva **clau privada de host** i envia la signatura al client. **El secret compartit no s'envia**: el client també l'ha calculat i pot obtenir el mateix resum.
2. El client comprova que la **clau pública de host és l'esperada**, comparant-la amb `known_hosts` o verificant-ne l'empremta si és la primera connexió.
3. El client utilitza aquesta clau pública per **verificar la signatura** del resum que ell mateix calcula.
4. Si totes dues comprovacions són correctes, el client pot donar el servidor per autèntic i continuar amb l'establiment del canal xifrat.

L'atacant pot conèixer la clau pública i observar la signatura, però no pot generar una signatura vàlida per a un intercanvi manipulat sense la clau privada del servidor. Tampoc no li serveix copiar una signatura anterior, perquè està vinculada a les dades d'aquell intercanvi.

**La clau privada signa; la clau pública verifica.** Aquestes claus de host autentiquen el servidor i són diferents de les claus de l'usuari que veurem després.

### Del secret compartit al xifratge de la sessió

A partir del secret `S` i de dades de l'intercanvi, SSH calcula diverses claus mitjançant una funció de derivació. **Derivar** vol dir obtenir les claus necessàries a partir d'aquest material compartit. Com que tots dos extrems tenen les mateixes dades, obtenen les mateixes claus sense enviar-les:

- una clau per xifrar el trànsit del client cap al servidor
- una clau per xifrar el trànsit del servidor cap al client
- segons l'algoritme negociat, claus per calcular codis d'autenticació de missatge (MAC), que permeten detectar modificacions del trànsit

Alguns modes de xifratge ja incorporen aquesta comprovació d'integritat i no necessiten un MAC separat.

**Diffie-Hellman permet acordar el secret; AES o ChaCha20 xifren les dades amb les claus derivades.**

Tot el trànsit posterior va xifrat amb algoritmes simètrics ràpids (AES, ChaCha20). La clau de host autentica l'intercanvi mitjançant la signatura; les claus simètriques derivades del secret compartit xifren el trànsit.

## Autenticació de l'usuari

Un cop establert el canal xifrat i autenticat el servidor, la persona que es connecta ha de demostrar que pot accedir al compte remot. Hi ha dos mètodes principals:

- **Contrasenya**: la persona introdueix la contrasenya del compte remot i el programa client SSH l'envia al servidor pel canal ja xifrat.
- **Clau pública**: la persona prepara un parell de claus d'autenticació al seu ordinador. En connectar-se, el programa client SSH signa una petició amb la clau privada i el servidor verifica la signatura amb la clau pública autoritzada per al compte remot.

**Acceptar `yes` en la primera connexió no genera claus d'usuari**: accepta la clau pública de host que presenta el servidor i la desa a `~/.ssh/known_hosts`. Aquesta comprovació de la identitat del servidor es fa tant si després accedim amb contrasenya com si ho fem amb clau pública. En tots dos casos, la sessió queda xifrada.

### Autenticació per clau pública, com funciona

Els passos següents corresponen a l'accés amb clau pública, com a alternativa a la contrasenya del compte remot:

1. La persona executa `ssh-keygen` al seu ordinador per generar un parell de claus d'autenticació. **No es generen automàticament en connectar-se**; normalment es preparen una vegada i es reutilitzen. Les claus queden a `~/.ssh/` del seu ordinador:
   - **Clau privada** (per exemple `~/.ssh/id_ed25519`): es conserva en aquest ordinador i no s'ha de compartir.
   - **Clau pública** (`~/.ssh/id_ed25519.pub`): es pot copiar al servidor.
2. La persona utilitza `ssh-copy-id` per copiar la clau pública al compte remot. Aquesta s'afegeix al fitxer `~/.ssh/authorized_keys` d'aquell compte, al servidor.
3. En una connexió posterior, un cop establert el canal xifrat i autenticat el servidor, el programa client SSH prepara una **petició d'autenticació vinculada a aquesta sessió** i la signa amb la clau privada de l'usuari.
4. El servidor SSH comprova que la clau pública està autoritzada per al compte remot i **verifica la signatura** amb aquesta clau.
5. Si les comprovacions són correctes, l'usuari queda autenticat i pot accedir al compte.

La signatura demostra que el client disposa de la clau privada corresponent, **sense enviar-la al servidor**. Com que està vinculada a la sessió, no es pot reutilitzar en una altra connexió.

En aquest mètode no s'envia cap contrasenya d'accés al servidor. Si la clau privada està protegida amb una frase de pas, aquesta s'utilitza localment per desbloquejar-la.

> **Les claus de host autentiquen el servidor; les claus de l'usuari permeten autenticar-se al compte remot. El trànsit es xifra amb les claus simètriques de sessió, tant si s'accedeix amb contrasenya com amb clau pública.**

### Algoritmes recomanats

- **ED25519**: modern, ràpid, curt i molt segur. És l'opció per defecte recomanada avui dia.
- **RSA de 4096 bits**: encara vàlid, però més lent i les claus són molt més llargues.
- **RSA de 1024 bits o inferiors**: **obsolets**, no fer servir.
- **DSA**: obsolet.

## SSH vs TLS: una comparativa

TLS (el protocol darrere de HTTPS, IMAPS, SMTPS, FTPS...) i SSH resolen el mateix problema —comunicació xifrada per una xarxa insegura— però amb models de confiança molt diferents. Aquesta comparativa serà especialment útil quan al mòdul entrem en HTTPS (RA5) i correu segur (RA4).

**HTTPS és HTTP sobre TLS. SSH té el seu propi protocol de seguretat i no utilitza TLS.**

### Diferències clau

La taula compara el model de confiança directa d'SSH que utilitzarem amb una connexió HTTPS habitual amb certificat.

| Aspecte | TLS | SSH |
|---|---|---|
| Model de confiança | Autoritats de Certificació (CA) externes | Confiança directa (tu decideixes) |
| Identitat del servidor | Certificat X.509 i prova de possessió de la privada | Clau pública de host i signatura de l'intercanvi |
| Comprovació de la destinació | El nom ha de coincidir amb el SAN del certificat | La clau ha de correspondre a la guardada per al nom o la IP |
| Primera connexió | Habitualment automàtica si el certificat és vàlid i de confiança | Cal verificar i acceptar la clau si encara no és coneguda |
| Intercanvi de claus | Habitualment ECDHE | DH o variants com ECDH |
| Autenticació de l'usuari | L'aplicació sol demanar credencials després d'establir TLS | SSH pot demanar contrasenya o verificar una signatura de l'usuari |
| Port típic | Diversos (443, 993, 465, ...) | 22 |
| Cas d'ús principal | Serveis d'aplicació (web, correu, ...) | Administració remota |

### CN i SAN (només TLS)

Quan al mòdul es parlarà de certificats (RA4 i RA5), apareixeran aquests dos conceptes:

- **CN (Common Name)**: el nom principal històric del certificat.
- **SAN (Subject Alternative Name)**: llista moderna de tots els noms vàlids per al certificat.

**Regla d'or**: els clients moderns (navegadors, Thunderbird, etc.) **ignoren el CN** i només validen el **SAN**. Si el nom que fem servir en connectar no apareix a la llista SAN, sortirà un avís de seguretat.

En el model d'SSH que utilitzarem no hi ha CN ni SAN: `known_hosts` associa la clau pública de host amb el nom o la IP del servidor.

### El handshake de TLS (per contrast)

En una connexió TLS moderna amb certificat, com una connexió HTTPS habitual, les funcions principals són:

1. Client i servidor negocien els algoritmes i intercanvien valors públics per obtenir un secret compartit, habitualment mitjançant ECDHE.
2. El servidor presenta el seu **certificat** i una signatura feta amb la privada corresponent.
3. El client comprova la cadena de confiança del certificat, la seva vigència i que el nom del servidor coincideix amb el SAN.
4. El client utilitza la pública del certificat per verificar la signatura, que vincula el servidor amb aquesta negociació.
5. A partir del secret compartit es deriven les claus simètriques que protegeixen el trànsit.

És un resum de les funcions del protocol: algunes comprovacions i operacions es fan durant el mateix intercanvi.

En una connexió SSH:

1. Client i servidor negocien els algoritmes i intercanvien els valors públics de Diffie-Hellman.
2. El servidor envia la seva **clau pública de host** i la signatura del resum de l'intercanvi, generada amb la seva **clau privada**.
3. El client comprova que la clau pública és l'esperada i verifica la signatura.
4. Es deriven les claus simètriques i s'activa el xifratge de la sessió.
5. L'usuari s'autentica a través del canal protegit.

En una frase:

> **En els exemples d'aquesta unitat, TLS comprova la identitat amb certificats i CA; SSH compara la clau del servidor amb la guardada a `known_hosts`. En tots dos casos, el servidor també demostra que disposa de la privada corresponent.**

## Comandes bàsiques d'SSH

### Connexió a un servidor

```bash
ssh usuari@ip_del_servidor
```

Si el port SSH és diferent de 22:

```bash
ssh -p 2222 usuari@ip_del_servidor
```

### Executar una comanda remota sense obrir sessió interactiva

```bash
ssh usuari@servidor 'uptime'
```

### Còpia segura de fitxers: `scp`

`scp` (*secure copy*) fa servir el mateix protocol SSH per copiar fitxers entre màquines. La sintaxi general és:

```bash
scp <origen> <destí>
```

On qualsevol dels dos (origen o destí) pot ser local o remot. Els remots es descriuen amb `usuari@host:ruta`.

Exemples:

```bash
# Del client al servidor
scp fitxer.txt usuari@servidor:/home/usuari/

# Del servidor al client (el punt final és el directori actual)
scp usuari@servidor:/etc/hostname .

# Copiar tot un directori (opció -r)
scp -r carpeta/ usuari@servidor:/tmp/
```

### Execució remota d'aplicacions gràfiques: X11 forwarding

Un truc molt útil de SSH és **X11 forwarding**: executar una aplicació gràfica al servidor però que es dibuixi a la pantalla del client. Serveix quan tenim un servidor sense entorn gràfic instal·lat i volem fer servir una eina concreta puntualment.

```bash
ssh -X usuari@servidor
```

Un cop dins, si executem per exemple `gedit` o `gimp`, la finestra apareixerà a la nostra pantalla local, però el procés corre al servidor.

Requeriments:

- Servidor: línia `X11Forwarding yes` a `/etc/ssh/sshd_config`.
- Client: entorn gràfic amb suport X11 (Linux amb X.org o XWayland ho porten de sèrie). Des de Windows, calen eines addicionals com [VcXsrv](https://sourceforge.net/projects/vcxsrv/) o [MobaXterm](https://mobaxterm.mobatek.net/).

### Túnels SSH (port forwarding)

SSH permet transportar connexions d'altres serveis a través del seu canal xifrat. Això es coneix com a **túnel SSH** o **port forwarding**.

Hi ha dos tipus principals:

#### Túnel local (`-L`)

```bash
ssh -L PORT_LOCAL:HOST_DESTÍ:PORT_DESTÍ usuari@servidor_ssh
```

Escolta al port local del client i envia tot el trànsit, a través de la connexió SSH, al `HOST_DESTÍ:PORT_DESTÍ` (accedit *des del servidor SSH*).

Exemple: accedir a un servei web intern d'una empresa que només es veu des del servidor de l'empresa:

```bash
ssh -L 8080:intranet.empresa.edu:80 usuari@servidor.empresa.edu
```

Ara, al navegador local, `http://localhost:8080` mostra el que veuria `intranet.empresa.edu:80` accedint des del servidor.

El recorregut és:

```text
Navegador → localhost:8080 → canal SSH → servidor SSH → intranet:80
```

Aquí `localhost:8080` és al nostre ordinador. El programa `ssh` hi rep les connexions i les transporta fins al servidor SSH, que es connecta a la intranet.

**SSH xifra el tram entre el client i el servidor SSH.** El tram del servidor SSH fins a la intranet depèn del protocol del servei: en aquest exemple és HTTP i no va xifrat amb TLS.

#### Túnel remot (`-R`)

Fa el contrari: obre un port al servidor SSH que reenvia trànsit cap al client. Útil per exposar temporalment un servei local a un servidor.

Per exemple, tenim un servei web al port `8000` del nostre ordinador i volem accedir-hi des del servidor:

```bash
ssh -N -R 9000:localhost:8000 usuari@servidor_ssh
```

Des del servidor, `http://localhost:9000` arriba al servei del nostre ordinador:

```text
Servidor: localhost:9000 → canal SSH → ordinador del client: localhost:8000
```

En aquesta ordre, `localhost:8000` es refereix a l'ordinador del client. Per defecte, el port remot només és accessible des del mateix servidor. `-N` indica que volem el túnel sense executar cap comanda remota.

Els túnels són especialment útils per protegir protocols que no tenen xifratge propi. Un exemple clàssic (que veurem a la pràctica): **VNC** dins un túnel SSH.

## Configuració del servidor SSH: `sshd_config`

El fitxer principal de configuració és `/etc/ssh/sshd_config`. Alguns paràmetres essencials:

| Paràmetre | Efecte |
|---|---|
| `Port 22` | Port on escolta el servei |
| `PermitRootLogin no` | Prohibeix l'entrada directa com a root |
| `PasswordAuthentication no` | Desactiva el mètode d'autenticació `password` |
| `PubkeyAuthentication yes` | Activa l'accés per clau pública |
| `AuthenticationMethods publickey` | Exigeix autenticar-se mitjançant clau pública |
| `MaxAuthTries 3` | Nombre màxim d'intents d'autenticació per connexió |
| `AllowUsers usuari1 usuari2` | Llista blanca d'usuaris que poden entrar |
| `X11Forwarding yes` | Permet reenviament X11 (aplicacions gràfiques) |
| `LoginGraceTime 30` | Temps màxim per completar el login (segons) |

`PasswordAuthentication no` no significa, per si sol, «només claus»: un altre mètode, l'autenticació interactiva, també pot demanar una contrasenya. Per exigir l'accés per clau pública al laboratori, utilitzarem `AuthenticationMethods publickey` amb `PubkeyAuthentication yes`.

Sempre que es modifiqui aquest fitxer:

1. Validar la sintaxi abans de reiniciar amb `sudo sshd -t`. Si no diu res, tot va bé.
2. Reiniciar el servei: `sudo systemctl restart ssh`.
3. **No tancar la sessió actual fins haver comprovat que una segona sessió pot entrar amb la nova configuració** — si no, ens podem quedar bloquejats fora del servidor.

### Bones pràctiques de securització

En un servidor real exposat a Internet, les mesures mínimes recomanades són:

- Autenticació **només per clau pública** (`AuthenticationMethods publickey` i `PubkeyAuthentication yes`), després d'haver comprovat que l'accés amb les claus funciona.
- **Prohibir root directe** (`PermitRootLogin no`) i entrar amb un usuari normal + `sudo`.
- **Limitar intents** amb `MaxAuthTries` i `LoginGraceTime`.
- **Llista blanca d'usuaris** amb `AllowUsers`.
- **Tallafoc** (UFW) que només obri el port SSH.
- **Fail2ban** per bloquejar automàticament IPs que fan intents fallits repetits.

### Nota sobre "canviar el port"

Canviar el port 22 per un altre pot reduir els intents automàtics que només proven el port habitual. Tanmateix, un escàner pot trobar el nou port: el canvi no reforça el xifratge ni l'autenticació.

És una mesura complementària que no substitueix les claus d'autenticació, les actualitzacions, el control d'accés, el tallafoc o eines com Fail2ban.

## Accés remot gràfic

Quan la línia de comandes no és suficient (per exemple, cal ensenyar alguna cosa a un usuari, cal accedir a una aplicació gràfica no consolable, o cal donar suport a un client Windows), s'utilitzen protocols d'escriptori remot.

### VNC (Virtual Network Computing)

VNC és un protocol multiplataforma antic i molt estès. Existeixen múltiples implementacions (TightVNC, RealVNC, TigerVNC, x11vnc, ...) que interoperen entre elles perquè comparteixen el protocol base **RFB (Remote Framebuffer)**.

Característiques:

- **Multiplataforma**: Linux, Windows, macOS, mòbils.
- **Simple**: el servidor comparteix la pantalla, el client la veu i envia teclat i ratolí.
- **La seguretat depèn de la implementació i la configuració**: l'autenticació VNC clàssica només utilitza els primers 8 caràcters de la contrasenya i no xifra la sessió. Algunes implementacions admeten proteccions addicionals. Al laboratori el protegirem amb un túnel SSH i no exposarem el port VNC directament a Internet.

La solució pràctica és **encapsular VNC dins un túnel SSH**: el servidor VNC escolta només a `localhost`, i el client s'hi connecta a través d'un `ssh -L`. Així s'aprofita el xifratge d'SSH i la seva autenticació forta.

Port habitual: 5900 (5901 per a la segona sessió, 5902 per la tercera...).

### RDP (Remote Desktop Protocol)

RDP és el protocol propietari de Microsoft. És el que s'utilitza a les edicions Pro/Enterprise/Server de Windows per permetre "Escriptori remot".

Característiques:

- **Optimitzat**: es dibuixa localment tant com pot, transmet només diferències, comprimeix bé el trànsit. Molt més eficient que VNC en xarxes lentes.
- **Xifrat** integrat i autenticació via credencials de Windows.
- **Reserves de llicència**: només les edicions Pro i superiors de Windows poden actuar com a **servidor** RDP. Totes poden actuar com a client.
- **Multiplataforma pel costat client**: Windows, Linux (Remmina, `xfreerdp`), macOS.

Per fer que un Linux sigui **servidor** RDP (i el puguem controlar des d'un client Windows), cal instal·lar el paquet `xrdp`:

```bash
sudo apt install xrdp
```

`xrdp` és un servidor que "tradueix" RDP entrant a una sessió gràfica local, típicament amb XFCE o similar.

### Clients recomanats

- **Remmina** (Linux): client universal amb suport a RDP, VNC, SPICE... Ja ve instal·lat a Ubuntu Desktop.
- **Escriptori remot de Windows** (`mstsc`): client RDP natiu del sistema.
- **TigerVNC viewer / Remote Viewer** (`remote-viewer`): clients VNC.

### Solucions propietàries: TeamViewer, AnyDesk

Al món del suport tècnic i l'usuari domèstic, són molt populars **TeamViewer** i **AnyDesk**. Funcionen amb un patró diferent: els dos ordinadors es connecten a un servidor intermedi de l'empresa (Núvol) i aquest els posa en contacte. Això té l'avantatge que no cal obrir ports ni configurar tallafocs; només cal passar un identificador i una contrasenya.

Punts a tenir en compte:

- **Propietari**: no són programari lliure, ni es basen en estàndards oberts.
- **Depenen d'un tercer**: si l'empresa cau o canvia les condicions, tot depèn d'ella.
- **Ús comercial restringit**: la versió gratuïta és només per a ús personal.
- **Registre requerit** per aprofitar totes les funcions.

Per l'entorn d'aquest mòdul (administració de servidors, laboratoris tancats) VNC/RDP són suficients. TeamViewer/AnyDesk s'inclouen aquí només com a **referència de comparació**, no s'utilitzaran a les pràctiques.

## Recursos i lectures addicionals

- [OpenSSH — documentació oficial](https://www.openssh.com/manual.html)
- [SSH Academy — guia extensa (en anglès)](https://www.ssh.com/academy/ssh)
- Manual local d'una comanda: `man ssh`, `man sshd_config`, `man ssh-keygen`, `man scp`.
- RFC 4251 — arquitectura del protocol SSH.
