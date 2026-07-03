# Serveis de xarxa — FTP

## Introducció

**FTP (File Transfer Protocol)** és un dels protocols més antics d'Internet: el primer RFC que el descriu data del **1971** i va ser estandarditzat el **1985** (RFC 959). Per posar-ho en context, HTTP (el protocol de la web) és de 1996.

FTP forma part de la **capa d'aplicació** de TCP/IP. La seva funció és la **transferència de fitxers** entre client i servidor. És un protocol **amb estat (*stateful*)**: manté oberta una sessió durant tota la interacció, cosa que el diferencia de HTTP, que és sense estat.

Un cop establerta la sessió, el client té un *prompt* per donar ordres al servidor: pujar, baixar, esborrar, renombrar fitxers, canviar de directori, etc.

Aquest RA cobreix la instal·lació i configuració de **`vsftpd`** (Very Secure FTP Daemon) a Ubuntu Server. Al **Projecte 1** del mòdul es reprendrà el tema amb un servidor més potent, **ProFTPD**.

## Model client-servidor: dos canals

Una particularitat de FTP és que utilitza **dues connexions TCP** entre client i servidor:

- **Canal de control** (port 21 al servidor): per on viatgen les ordres i les respostes. Es manté obert tota la sessió.
- **Canal de dades** (port 20 o negociat, segons el mode): s'obre quan cal transferir un fitxer o el llistat d'un directori, i es tanca en acabar.

Aquesta separació és la que fa que FTP tingui **dos modes de funcionament** — actiu i passiu — que veurem tot seguit.

## Mode actiu vs. mode passiu

En **mode actiu** el client obre el canal de control cap al port 21 del servidor. Quan cal transferir dades, és el **servidor** qui obre la connexió de dades cap a un port que el client li ha indicat. Aquesta connexió entrant al client sol ser **bloquejada per tallafocs i NAT**, cosa que fa que el mode actiu sigui problemàtic en molts entorns.

En **mode passiu (PASV)** el servidor no inicia cap connexió cap al client. En lloc d'això, obre un **port aleatori** al seu costat i li'n dona el número al client. El client és qui obre les dues connexions (control i dades) cap al servidor. Aquesta és la forma **estàndard avui dia** perquè travessa millor NAT i tallafocs.

> Aquest tema s'explica en detall al vídeo que us passaré junt amb aquest material. Mireu-lo abans de fer les pràctiques.

## Diferència FTP / FTPS / SFTP

Sovint es confonen. Són tres coses diferents:

| Protocol | Base | Xifratge | Port habitual | Autenticació |
|---|---|---|---|---|
| **FTP** | FTP pur | Cap (text pla) | 21 | Usuari/contrasenya (en pla) |
| **FTPS** | FTP + TLS/SSL | Certificat X.509 (com HTTPS) | 21 (explícit) o 990 (implícit) | Usuari/contrasenya (xifrada per TLS) |
| **SFTP** | Subsistema d'**SSH** | Claus SSH | 22 (mateix que SSH) | Contrasenya o claus SSH |

Punts clau que sovint es malinterpreten:

- **FTPS** és FTP de tota la vida amb una capa TLS al damunt. Manté els dos canals (control i dades), l'estat, les comandes (`RETR`, `STOR`, `PASV`...). Només afegeix xifratge.
- **SFTP** **no té res a veure amb FTP**. És un protocol completament diferent, dissenyat des de zero, que es transporta dins d'una connexió **SSH**. Reutilitza tot el que ja fa SSH (autenticació, xifratge, integritat), amb un únic port.
- Per a **FTPS** cal un certificat generat amb **`openssl`**.
- Per a **SFTP** el servidor és el mateix **`openssh-server`** — no cal instal·lar res més; el subsistema SFTP ja ve activat per defecte a `/etc/ssh/sshd_config`.

En aquest RA farem servir **totes dues opcions** com a pràctica de xifratge (FTPS amb vsftpd, SFTP amb OpenSSH).

## `vsftpd`: instal·lació

**`vsftpd`** (Very Secure FTP Daemon) és un servidor FTP lleuger i orientat a seguretat, molt utilitzat a distribucions Linux. És l'eina que farem servir a les pràctiques d'aquest RA.

```bash
sudo apt update
sudo apt install vsftpd
```

En instal·lar-lo:

- Es crea el fitxer de configuració a **`/etc/vsftpd.conf`**.
- Es crea l'usuari del sistema `ftp` i el grup `ftp`.
- El servei s'inicia com a `vsftpd.service`.

```bash
sudo systemctl status vsftpd
sudo systemctl restart vsftpd
```

Els logs es consulten amb `journalctl -u vsftpd` o a `/var/log/vsftpd.log` (si està activat a `xferlog_file`).

## Fitxer de configuració: `/etc/vsftpd.conf`

El fitxer que crea `apt` ja porta molts exemples comentats. **Feu-ne una còpia abans de tocar-lo**:

```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bck
```

Directives més habituals (les principals):

| Directiva | Què fa |
|---|---|
| `listen=YES` | Fa que vsftpd escolti com a dimoni independent (no via `inetd`/`xinetd`). |
| `anonymous_enable=YES/NO` | Permet o denega el login anònim. |
| `local_enable=YES/NO` | Permet o denega el login d'usuaris del sistema. |
| `write_enable=YES/NO` | Permet operacions d'escriptura (`PUT`, `MKDIR`, `DELETE`...). |
| `local_umask=022` | Umask amb què es creen els fitxers pujats. |
| `chroot_local_user=YES` | Engabia els usuaris locals al seu directori (`chroot`). |
| `chroot_list_enable=YES` | Activa la llista d'excepcions al `chroot`. |
| `chroot_list_file=/etc/vsftpd.chroot_list` | Fitxer amb els usuaris que **no** s'han d'engabiar. |
| `allow_writeable_chroot=YES` | Permet que un directori engabiat sigui també d'escriptura. |
| `anon_root=/var/ftp/pub` | Directori on aterren els anònims. |
| `pasv_enable=YES` | Activa el mode passiu (per defecte ho està). |
| `pasv_min_port` / `pasv_max_port` | Rang de ports per al mode passiu (útil amb tallafocs). |
| `ssl_enable=YES` | Activa FTPS (TLS). |
| `rsa_cert_file` / `rsa_private_key_file` | Rutes al certificat i la clau privada. |

Després de qualsevol canvi:

```bash
sudo systemctl restart vsftpd
```

Si el servei no arrenca, es pot cercar l'error executant vsftpd manualment:

```bash
sudo vsftpd /etc/vsftpd.conf
```

## Accés anònim

L'usuari `anonymous` (àlies `ftp`) és una convenció clàssica per permetre descàrregues públiques sense credencials. **Mai** ha de tenir permís d'escriptura.

Estructura habitual:

```bash
sudo mkdir -p /var/ftp/pub
sudo chmod 555 /var/ftp/pub          # r-x per a tothom, sense escriptura
sudo chgrp ftp /var/ftp /var/ftp/pub
```

I al `vsftpd.conf`:

```
anonymous_enable=YES
anon_root=/var/ftp/pub
```

## Usuaris locals

Els usuaris locals són usuaris del sistema (definits a `/etc/passwd`). Si volem que **només accedeixin per FTP i no puguin iniciar sessió al sistema**, els posem el shell `/usr/sbin/nologin`.

```bash
sudo useradd -g ftp -d /home/ftp1 -m -s /usr/sbin/nologin ftp1
sudo passwd ftp1
```

- `-g ftp` — el fem membre del grup `ftp`.
- `-d /home/ftp1 -m` — crea el directori home ara mateix.
- `-s /usr/sbin/nologin` — shell de "no login" (no pot obrir sessió SSH ni consola local).

Perquè el sistema accepti `nologin` com a shell vàlid per a FTP, cal afegir-lo a `/etc/shells`:

```
/usr/sbin/nologin
```

> **Compte**: si l'usuari té `nologin`, tampoc podrà accedir per **SFTP** (que és SSH). Per a la pràctica d'SFTP haurem d'usar usuaris amb un shell real (`/bin/bash`), o configurar SSH amb `ForceCommand internal-sftp` (fora de l'abast d'aquest RA — es veu a RA6).

## Engabiar amb `chroot`

Per defecte, un usuari FTP pot navegar amunt del seu `home` i explorar el sistema de fitxers. Això és un problema de seguretat evident. La solució és el **`chroot`** ("change root"): fer creure a l'usuari que la seva arrel (`/`) és el seu propi directori. No pot pujar més amunt.

Configuració a `vsftpd.conf`:

```
chroot_local_user=YES                        # tots els locals engabiats per defecte
chroot_list_enable=YES                       # activa la llista d'excepcions
chroot_list_file=/etc/vsftpd.chroot_list     # llista d'usuaris que sí que poden sortir
allow_writeable_chroot=YES                   # per poder escriure dins de la gàbia
write_enable=YES
local_umask=022
```

I es crea el fitxer amb els usuaris "lliures" (un per línia):

```bash
sudo touch /etc/vsftpd.chroot_list
```

## FTPS: xifratge amb TLS

Perquè FTP viatgi xifrat, vsftpd pot afegir una capa TLS. Cal generar un certificat autofirmat amb OpenSSL i indicar-ne les rutes al `vsftpd.conf`.

```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
    -keyout /etc/ssl/private/vsftpd.key \
    -out /etc/ssl/certs/vsftpd.crt
sudo chmod 600 /etc/ssl/private/vsftpd.key
```

I a `vsftpd.conf`:

```
ssl_enable=YES
rsa_cert_file=/etc/ssl/certs/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1_2=YES
ssl_sslv2=NO
ssl_sslv3=NO
```

Per connectar-s'hi cal un client compatible amb FTPS explícit — el client `ftp` de línia de comandes clàssic **no ho és**, però `lftp`, `curl` i **FileZilla** sí.

## SFTP: xifratge via SSH

SFTP no és una variant de FTP: és un **subsistema d'SSH**. Si ja tenim `openssh-server` instal·lat, SFTP ja funciona sense fer res:

```bash
sudo apt install openssh-server
```

Comprovació:

```bash
sftp usuari@ip_del_servidor
```

L'usuari que es connecta per SFTP ha de tenir un shell vàlid (`/bin/bash`, per exemple). El fitxer `/etc/ssh/sshd_config` sol tenir aquesta línia activada per defecte:

```
Subsystem sftp /usr/lib/openssh/sftp-server
```

## Comandes FTP essencials

Un cop dins de la sessió FTP (`ftp>` o `sftp>`), les comandes principals són pràcticament idèntiques a Linux. **N'hi ha moltes més** (fàcilment 50-60), però amb aquestes 20 en tens prou per moure't. Consulta `help` dins del client per veure la llista completa d'un servidor concret.

Les que **has de saber per l'examen**:

| Comanda | Ús |
|---|---|
| `open <host>` | Connecta amb el servidor. |
| `bye` / `quit` | Tanca la sessió i surt del client. |
| `close` / `disconnect` | Tanca la sessió però no surt del client. |
| `pwd` | Directori de treball al servidor. |
| `cd` | Canvia de directori al servidor. |
| `lcd` | Canvia de directori al client (LOCAL). |
| `ls` / `dir` | Llista contingut del directori remot. |
| `get <fitxer>` | Descarrega un fitxer del servidor. |
| `mget <patró>` | Descarrega diversos fitxers (`mget *.txt`). |
| `put <fitxer>` | Puja un fitxer al servidor. |
| `mput <patró>` | Puja diversos fitxers. |
| `mkdir <nom>` | Crea directori remot. |
| `rmdir <nom>` | Esborra directori remot (ha d'estar buit). |
| `delete <fitxer>` | Esborra fitxer remot. |
| `mdelete <patró>` | Esborra diversos fitxers remots. |
| `mdir` / `mls` | Llista contingut de diversos directoris remots. |
| `rename <a> <b>` | Renombra un fitxer remot. |
| `ascii` | Mode de transferència text. |
| `binary` | Mode de transferència binari (per defecte). |
| `help` / `?` | Ajuda dins del client. |

## FileZilla: client gràfic

`FileZilla` és un client gràfic multiplataforma (Linux, Windows, macOS) que suporta **FTP**, **FTPS** i **SFTP**. És l'estàndard *de facto* al món educatiu i sysadmin.

```bash
sudo apt install filezilla
```

Camps de la barra ràpida:

- **Host**: IP o nom del servidor. Amb prefix `ftp://`, `ftps://` o `sftp://` s'indica el protocol.
- **Nom d'usuari** / **Contrasenya**.
- **Port**: 21 per FTP/FTPS, 22 per SFTP.

Per a configuracions més complexes s'utilitza el **Gestor de llocs** (`File → Site Manager`), on cada connexió es desa amb el seu protocol, credencials i opcions de xifratge.

A cada pràctica d'aquest RA farem les tasques **primer per línia de comandes** (per fixar les comandes) i **després verificarem el resultat amb FileZilla** (per confirmar visualment).

## Referències

- Documentació oficial de vsftpd: [security.appspot.com/vsftpd.html](https://security.appspot.com/vsftpd.html)
- Documentació d'Ubuntu sobre FTP: [ubuntu.com/server/docs/service-ftp](https://ubuntu.com/server/docs/service-ftp)
- FileZilla: [filezilla-project.org](https://filezilla-project.org/)
- RFC 959 — File Transfer Protocol.
- RFC 4217 — Securing FTP with TLS (FTPS).
- RFC 4251/4253 — SSH (base de SFTP).
