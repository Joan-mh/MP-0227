# Serveis de xarxa — Accés remot i SSH

## Per què necessitem accés remot

Quan un servidor està en un rack, en un CPD o simplement en una màquina virtual d'un laboratori, no ens és pràctic anar-hi físicament cada vegada que vulguem administrar-lo. L'**accés remot** permet controlar un ordinador des d'un altre a través de la xarxa, com si estiguéssim asseguts davant seu.

Hi ha dos grans blocs d'eines d'accés remot:

- **Accés en mode text (terminal remot)** — l'administrador rep una consola de comandes del servidor. És lleuger, ràpid, i cobreix el 95% de la feina d'administració. L'estàndard actual és **SSH**.
- **Accés en mode gràfic (escriptori remot)** — l'administrador rep el mateix escriptori que veuria en pantalla. Necessari per clients Windows o per usuaris sense pràctica en línia de comandes. Els protocols habituals són **VNC** i **RDP**.

### Protocols insegurs i protocols segurs

Els protocols antics d'accés remot (`telnet`, `rlogin`, `rsh`, `ftp`) transmetien tot el trànsit —incloent contrasenyes— **en clar** per la xarxa. Qualsevol amb accés al mateix segment de xarxa podia llegir-ho amb un capturador de paquets com Wireshark. Avui dia estan pràcticament abandonats.

Els substituts moderns (**SSH, SFTP, HTTPS, IMAPS, SMTPS**...) **xifren** el trànsit d'extrem a extrem. El xifratge es fa amb una combinació de criptografia asimètrica (per posar-se d'acord en una clau) i simètrica (per xifrar el trànsit real, molt més ràpid).

## SSH: el model de confiança

SSH (*Secure Shell*) és el protocol estàndard per administrar servidors Linux/Unix des de fa dècades. Escolta al port **22/TCP** per defecte.

Cal entendre un punt fonamental abans d'anar més enllà: **SSH no fa servir certificats X.509 ni Autoritats de Certificació (CA)**. La seva seguretat es basa en un model diferent que en veurem en detall.

### Cifrar ≠ tenir un certificat

Aquest és un dels malentesos més freqüents:

- **Xifrar** és una **acció**: transformar dades llegibles en il·legibles amb una clau.
- Un **certificat** és un document digital signat per una autoritat que vincula una **clau pública** amb una identitat (un nom de servidor, en el cas de TLS).

El certificat en si no xifra res: només serveix perquè el client sàpiga que la clau pública que rep pertany realment a qui diu ser. Un cop verificada la identitat, es fa servir la clau pública per acordar una clau simètrica de sessió, i aquesta última és la que xifra el trànsit.

**SSH salta el pas del certificat i l'autoritat de certificació**. En comptes d'això, es basa en **confiança directa**: la primera vegada que et connectes a un servidor, la seva clau pública queda guardada al fitxer `~/.ssh/known_hosts` del client. A partir d'aquell moment, cada connexió comprova que la clau del servidor no ha canviat.

### Identitat del servidor SSH

Cada servidor SSH té un parell de claus generat en instal·lar el paquet `openssh-server`:

- La **clau privada de host** roman al servidor, dins `/etc/ssh/` (`ssh_host_rsa_key`, `ssh_host_ed25519_key`...).
- La **clau pública de host** s'envia al client en cada connexió.

No hi ha validació per nom (no hi ha DNS, CN, SAN, etc. com a TLS). El client simplement decideix si confia o no en la clau que rep.

### La primera connexió

En la primera connexió a un servidor SSH que no coneixem, veurem un missatge així:

```
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:xyz123abc...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

El client ens està dient: *no puc verificar automàticament aquesta clau; decideix tu si te'n refies*. Si contestem `yes`, la clau queda guardada a `~/.ssh/known_hosts` i totes les connexions posteriors la compararan amb l'emmagatzemada.

Aquest fitxer `known_hosts` actua, en la pràctica, com una **CA local manual**: som nosaltres qui decidim en què confiem, no una autoritat externa.

### Connexions posteriors i atac Man-in-the-Middle (MITM)

En les connexions següents al mateix servidor:

1. El servidor envia la seva clau pública de host.
2. El client la compara amb la guardada a `known_hosts`.
3. Si coincideix → connexió segura, sense preguntar res.
4. Si **no** coincideix → error crític, la connexió es rebutja.

Un canvi inesperat de clau **no és normal**. Vol dir una de dues coses: o bé el servidor s'ha reinstal·lat (i cal esborrar l'entrada antiga del `known_hosts`), o bé algú s'ha col·locat entre el client i el servidor real i està intentant fer-se passar per ell — això és el que s'anomena atac **Man-in-the-Middle (MITM)**.

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

Un cop verificada la identitat del servidor, cal establir una clau simètrica compartida per xifrar la sessió. Aquí entra en joc l'algoritme **Diffie-Hellman (DH)**, un dels grans invents de la criptografia moderna (1976).

### La idea fonamental

Diffie-Hellman permet que **dues parts es posin d'acord en una clau secreta comuna** sense que ningú que estigui escoltant la comunicació la pugui deduir. Sembla impossible, però funciona gràcies a operacions matemàtiques que són fàcils de fer en una direcció i inviables de revertir.

### Pas a pas

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

### Per què és segur

Un atacant que escolta tot el trànsit veu `p`, `g`, `A` i `B`, però per deduir `a` a partir de `A = g^a mod p` hauria de resoldre el que en matemàtiques s'anomena el **problema del logaritme discret**: donat `A`, `g` i `p`, trobar `a`. Amb primers de 2048 bits o més, aquest problema és computacionalment inviable (segles de càlcul amb els ordinadors actuals).

### Del secret compartit al xifratge de la sessió

A partir del valor `S`, SSH deriva diverses claus simètriques:

- una clau per xifrar el trànsit del client cap al servidor
- una clau per xifrar el trànsit del servidor cap al client
- claus per calcular codis d'autenticació de missatge (MAC) que asseguren la integritat

Tot el trànsit posterior va xifrat amb algoritmes simètrics ràpids (AES, ChaCha20). La clau pública del host **no xifra dades**; només s'utilitza per **signar l'intercanvi Diffie-Hellman** i evitar així que un atacant MITM pugui interposar-s'hi (perquè no té la clau privada del servidor per signar amb ella).

## Autenticació de l'usuari

Un cop el canal està xifrat, l'usuari ha de demostrar qui és. Hi ha dos mètodes principals:

- **Contrasenya**: el client envia la contrasenya pel canal ja xifrat. Simple però vulnerable a atacs de força bruta i a contrasenyes febles.
- **Clau pública**: mètode fortament recomanat. L'usuari genera un parell de claus al client (una privada que es queda al client, una pública que es puja al servidor). En autenticar-se, el servidor llança un repte matemàtic que només es pot resoldre amb la clau privada.

### Autenticació per clau pública, com funciona

1. L'usuari genera un parell de claus al client amb `ssh-keygen`. Les claus queden a `~/.ssh/`:
   - clau privada (per exemple `~/.ssh/id_ed25519`), mai s'ha de compartir.
   - clau pública (`~/.ssh/id_ed25519.pub`), es pot compartir sense problema.
2. La clau pública es copia al servidor amb `ssh-copy-id`, que l'afegeix al fitxer `~/.ssh/authorized_keys` de l'usuari destí.
3. En connectar, el servidor llança un repte xifrat amb la clau pública. Només qui tingui la clau privada pot desxifrar-lo i respondre correctament.

Cap contrasenya viatja per la xarxa, i la clau privada **no surt mai del client**.

### Algoritmes recomanats

- **ED25519**: modern, ràpid, curt i molt segur. És l'opció per defecte recomanada avui dia.
- **RSA de 4096 bits**: encara vàlid, però més lent i les claus són molt més llargues.
- **RSA de 1024 bits o inferiors**: **obsolets**, no fer servir.
- **DSA**: obsolet.

## SSH vs TLS: una comparativa

TLS (el protocol darrere de HTTPS, IMAPS, SMTPS, FTPS...) i SSH resolen el mateix problema —comunicació xifrada per una xarxa insegura— però amb models de confiança molt diferents. Aquesta comparativa serà especialment útil quan al mòdul entrem en HTTPS (RA5) i correu segur (RA4).

### Diferències clau

| Aspecte | TLS | SSH |
|---|---|---|
| Model de confiança | Autoritats de Certificació (CA) externes | Confiança directa (tu decideixes) |
| Identitat del servidor | Certificat X.509 amb SAN/CN | Clau pública de host |
| Validació per nom de domini | Sí (SAN ha de coincidir) | No |
| Primera connexió | Automàtica (si la CA és de confiança) | Interactiva (`yes/no`) |
| Intercanvi de claus | Certificat + DH efímer (ECDHE) | Diffie-Hellman |
| Autenticació del client | Rarament (només casos especials) | Sovint (claus públiques) |
| Port típic | Diversos (443, 993, 465, ...) | 22 |
| Cas d'ús principal | Serveis d'aplicació (web, correu, ...) | Administració remota |

### CN i SAN (només TLS)

Quan al mòdul es parlarà de certificats (RA4 i RA5), apareixeran aquests dos conceptes:

- **CN (Common Name)**: el nom principal històric del certificat.
- **SAN (Subject Alternative Name)**: llista moderna de tots els noms vàlids per al certificat.

**Regla d'or**: els clients moderns (navegadors, Thunderbird, etc.) **ignoren el CN** i només validen el **SAN**. Si el nom que fem servir en connectar no apareix a la llista SAN, sortirà un avís de seguretat.

SSH no té CN ni SAN: no valida noms, valida claus.

### El handshake de TLS (per contrast)

En una connexió TLS:

1. El servidor envia el seu **certificat**.
2. El client comprova que el certificat està signat per una CA en la qual confia (té la seva clau pública al magatzem de certificats).
3. El client comprova que el nom (SAN) del certificat coincideix amb el nom pel qual s'ha connectat.
4. El client extreu la **clau pública** del certificat.
5. S'utilitza Diffie-Hellman (concretament ECDHE) per acordar una clau simètrica.
6. A partir d'aquí, el trànsit va xifrat simètricament.

En una connexió SSH:

1. El servidor envia la seva **clau pública de host**.
2. El client la compara amb `~/.ssh/known_hosts` (o pregunta si és la primera vegada).
3. Es fa Diffie-Hellman signat amb la clau del servidor.
4. Trànsit xifrat simètricament.

En una frase:

> **TLS confia en tercers (les CA). SSH confia en la memòria (el `known_hosts`).**

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

Un dels superpoders d'SSH és poder redirigir ports a través de la connexió xifrada. Això es coneix com a **túnel SSH** o **port forwarding**.

Hi ha dos tipus principals:

#### Túnel local (`-L`)

```bash
ssh -L PORT_LOCAL:HOST_DESTÍ:PORT_DESTÍ usuari@servidor_ssh
```

Escolta al port local del client i envia tot el trànsit, a través de la connexió SSH, al `HOST_DESTÍ:PORT_DESTÍ` (accedit *des del servidor SSH*).

Exemple: accedir a un servei web intern d'una empresa que només es veu des del servidor de l'empresa:

```bash
ssh -L 8080:intranet.empresa.local:80 usuari@servidor.empresa.local
```

Ara, al navegador local, `http://localhost:8080` mostra el que veuria `intranet.empresa.local:80` accedint des del servidor.

#### Túnel remot (`-R`)

Fa el contrari: obre un port al servidor SSH que reenvia trànsit cap al client. Útil per exposar temporalment un servei local a un servidor.

Els túnels són especialment útils per protegir protocols que no tenen xifratge propi. Un exemple clàssic (que veurem a la pràctica): **VNC** dins un túnel SSH.

## Configuració del servidor SSH: `sshd_config`

El fitxer principal de configuració és `/etc/ssh/sshd_config`. Alguns paràmetres essencials:

| Paràmetre | Efecte |
|---|---|
| `Port 22` | Port on escolta el servei |
| `PermitRootLogin no` | Prohibeix l'entrada directa com a root |
| `PasswordAuthentication no` | Desactiva l'accés amb contrasenya (només claus) |
| `PubkeyAuthentication yes` | Activa l'accés per clau pública |
| `MaxAuthTries 3` | Nombre màxim d'intents d'autenticació per connexió |
| `AllowUsers usuari1 usuari2` | Llista blanca d'usuaris que poden entrar |
| `X11Forwarding yes` | Permet reenviament X11 (aplicacions gràfiques) |
| `LoginGraceTime 30` | Temps màxim per completar el login (segons) |

Sempre que es modifiqui aquest fitxer:

1. Validar la sintaxi abans de reiniciar amb `sudo sshd -t`. Si no diu res, tot va bé.
2. Reiniciar el servei: `sudo systemctl restart ssh`.
3. **No tancar la sessió actual fins haver comprovat que una segona sessió pot entrar amb la nova configuració** — si no, ens podem quedar bloquejats fora del servidor.

### Bones pràctiques de securització

En un servidor real exposat a Internet, les mesures mínimes recomanades són:

- Autenticació **només per clau pública** (`PasswordAuthentication no`).
- **Prohibir root directe** (`PermitRootLogin no`) i entrar amb un usuari normal + `sudo`.
- **Limitar intents** amb `MaxAuthTries` i `LoginGraceTime`.
- **Llista blanca d'usuaris** amb `AllowUsers`.
- **Tallafoc** (UFW) que només obri el port SSH.
- **Fail2ban** per bloquejar automàticament IPs que fan intents fallits repetits.

### Nota sobre "canviar el port"

Un consell clàssic és canviar el port 22 per un altre (per exemple 2022) per "amagar" el servei dels atacs automàtics. En la pràctica moderna això es considera **security by obscurity**: no aporta seguretat real (un escàner de ports troba el servei igualment) i pot causar problemes (regles de tallafoc, monitorització, documentació confusa).

Les mesures que **sí** aporten seguretat real són les de la llista anterior: **claus públiques + `fail2ban`**. Aquestes són efectives contra el 99% dels atacs automàtics; canviar el port només filtra els més ximples.

## Accés remot gràfic

Quan la línia de comandes no és suficient (per exemple, cal ensenyar alguna cosa a un usuari, cal accedir a una aplicació gràfica no consolable, o cal donar suport a un client Windows), s'utilitzen protocols d'escriptori remot.

### VNC (Virtual Network Computing)

VNC és un protocol multiplataforma antic i molt estès. Existeixen múltiples implementacions (TightVNC, RealVNC, TigerVNC, x11vnc, ...) que interoperen entre elles perquè comparteixen el protocol base **RFB (Remote Framebuffer)**.

Característiques:

- **Multiplataforma**: Linux, Windows, macOS, mòbils.
- **Simple**: el servidor comparteix la pantalla, el client la veu i envia teclat i ratolí.
- **Insegur per defecte**: la contrasenya original està **limitada a 8 caràcters** i el trànsit no va xifrat. Per això mai s'exposa VNC directament a Internet.

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
