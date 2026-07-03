# Pràctiques RA3 — FTP

> Farem servir **`vsftpd`** com a servidor. La xarxa és `192.168.X.0/24` — el tercer octet `X` el trieu vosaltres (mantingueu-lo coherent a totes les parts).
>
> A cada part farem les operacions **primer per línia de comandes** (client `ftp`/`sftp`) i **després les verificarem amb FileZilla**. Windows com a client és opcional — si voleu comprovar que FileZilla també hi funciona, podeu, però no és obligatori.

## Part 1 — Instal·lació i configuració bàsica de vsftpd (2 hores)

### Objectius

- Instal·lar `vsftpd` a Ubuntu Server.
- Configurar accés **anònim** de només lectura.
- Crear **3 usuaris locals** amb permisos diferenciats.
- Verificar l'accés per **línia de comandes i FileZilla**.

### Tasques

#### 1. Instal·lació

1. Actualitza els repositoris i instal·la `vsftpd`.
2. Comprova que el servei arrenca correctament i escolta al port 21 (`ss -tulpn | grep :21`).
3. Instal·la també `filezilla` al client Linux (i, si vols, també al Windows).

#### 2. Accés anònim

1. Crea la ruta `/var/ftp/pub` amb permisos `555` i propietat de grup `ftp`.
2. Deixa dins un parell de fitxers de prova (`.txt` amb qualsevol contingut).
3. Modifica `/etc/vsftpd.conf` perquè:
   - Permeti login anònim (`anonymous_enable=YES`).
   - L'arrel de l'anònim sigui `/var/ftp/pub` (`anon_root=...`).
   - **No** permeti escriptura anònima.
4. Reinicia el servei.

#### 3. Verificació de l'anònim

Per **línia de comandes**:

```bash
ftp <IP_SERVIDOR>
# Usuari: anonymous  (o ftp)
# Contrasenya: (buida)
ls
get <fitxer>
put nouintent.txt        # ha de fallar amb "550 Permission denied"
bye
```

Per **FileZilla**: connexió amb `Host: ftp://<IP_SERVIDOR>`, sense credencials. Comprova que veus els fitxers i que no pots pujar-ne.

#### 4. Usuaris locals

Crea **tres** usuaris locals: `ftp1`, `ftp2`, `ftp3`. Tots del grup `ftp`, amb `nologin` com a shell i el seu home a `/home/ftpN`. Assigna'ls contrasenya.

Recorda afegir `/usr/sbin/nologin` a `/etc/shells` perquè vsftpd els accepti.

#### 5. Configuració per als locals

Modifica `/etc/vsftpd.conf` perquè `local_enable=YES` i `write_enable=YES`. Reinicia el servei.

#### 6. Verificació dels locals

Per a cada usuari (`ftp1`, `ftp2`, `ftp3`), primer per CLI i després amb FileZilla:

1. Connecta-t'hi.
2. Comprova que aterres al seu `home`.
3. Puja un fitxer, comprova que hi és, esborra'l.
4. **Intenta pujar més amunt del home** (`cd ..`) — a aquesta part encara pot sortir (fins que l'engabiem a la Part 2).

### Resultats esperats

- L'anònim pot llistar i descarregar, però no pujar res.
- Els 3 usuaris locals poden pujar i baixar als seus homes.
- Tot funciona per CLI i verificat visualment amb FileZilla.

---

## Part 2 — Xifratge (FTPS + SFTP) i engabiar amb chroot (2 hores)

### Objectius

- Engabiar (`chroot`) els usuaris locals al seu directori home.
- Afegir una llista d'excepcions per deixar-ne un fora de la gàbia.
- Configurar **FTPS** (FTP sobre TLS) amb un certificat autofirmat.
- Configurar **SFTP** amb OpenSSH i entendre que **no és FTP**.
- Verificar les tres connexions (FTP, FTPS, SFTP) amb CLI i FileZilla.

### Tasques

#### 1. Engabiar tots els usuaris locals excepte un

A `/etc/vsftpd.conf`:

```
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd.chroot_list
allow_writeable_chroot=YES
```

Crea `/etc/vsftpd.chroot_list` i posa-hi només `ftp1` (aquest serà l'usuari lliure). Reinicia.

#### 2. Verificació de la gàbia

1. Connecta't com a `ftp2` per CLI i intenta `cd ..` diverses vegades. No has de poder sortir del seu home.
2. Connecta't com a `ftp1` per CLI i comprova que sí que pots navegar per sobre.
3. Repeteix amb FileZilla — el panell remot ha de mostrar `/` com el home de l'usuari engabiat.

#### 3. Generar el certificat per a FTPS

```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
    -keyout /etc/ssl/private/vsftpd.key \
    -out /etc/ssl/certs/vsftpd.crt
sudo chmod 600 /etc/ssl/private/vsftpd.key
```

Al *Common Name (CN)* posa la IP del teu servidor.

#### 4. Activar FTPS

Afegeix al `vsftpd.conf`:

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

Reinicia.

#### 5. Verificació de FTPS

El client `ftp` clàssic no suporta FTPS. Fes servir **`lftp`** o **FileZilla**:

Amb `lftp`:

```bash
lftp -u ftp2 ftps://<IP_SERVIDOR>
```

Amb **FileZilla**: `Host: ftps://<IP>`, protocol `FTP - File Transfer Protocol`, xifratge `Require explicit FTP over TLS`. Accepta el certificat autofirmat quan aparegui.

Comprova que ara **amb FTP pur ja no et deixa fer login** (perquè `force_local_logins_ssl=YES`).

#### 6. Configurar SFTP amb OpenSSH

Aquest apartat demostra que SFTP no té res a veure amb vsftpd — és un servei apart.

1. Instal·la `openssh-server` si no hi és.
2. Comprova que el subsistema SFTP està activat al `sshd_config` (línia `Subsystem sftp ...`).
3. Els usuaris que faran servir SFTP necessiten un **shell real** (no `nologin`). Canvia el shell de `ftp1` a `/bin/bash`:

```bash
sudo usermod -s /bin/bash ftp1
```

#### 7. Verificació d'SFTP

Per CLI:

```bash
sftp ftp1@<IP_SERVIDOR>
ls
put fitxer.txt
bye
```

Amb **FileZilla**: `Host: sftp://<IP>`, port 22, usuari `ftp1`. La primera connexió et demanarà d'acceptar la *host key* del servidor.

### Resultats esperats

- `ftp2` i `ftp3` són engabiats; `ftp1` no.
- FTPS funciona amb `lftp` o FileZilla; FTP pur ja no deixa autenticar.
- SFTP funciona per `ftp1` per l'OpenSSH, port 22, completament separat del vsftpd.

---

## Part 3 — Ús intensiu de comandes i verificació de permisos (3 hores)

> Aquesta part és més substancial: hi juguem amb comandes reals, estructura de directoris, permisos diferenciats i la comparativa CLI vs. FileZilla. Preneu-vos-la seriosament — sortiran preguntes de l'examen d'aquí.

### Objectius

- Practicar les **comandes essencials** de FTP i SFTP.
- Muntar una estructura de directoris amb permisos que diferencien els 3 usuaris.
- **Verificar cada operació primer per CLI i després amb FileZilla**.
- Documentar els errors que es reben quan un usuari intenta una operació que no té permesa.

### Preparació del servidor

Al servidor, crea aquesta estructura fora dels homes:

```bash
sudo mkdir -p /srv/ftp/docs /srv/ftp/multimedia /srv/ftp/projectes
sudo chown -R root:ftp /srv/ftp
sudo chmod 755 /srv/ftp/docs           # lectura per a tothom
sudo chmod 775 /srv/ftp/multimedia     # només ftp2 pot escriure
sudo chmod 775 /srv/ftp/projectes      # només ftp1 pot escriure
```

I fes que:

- **`ftp1`** sigui propietari d'`/srv/ftp/projectes` (lectura + escriptura allà).
- **`ftp2`** sigui propietari d'`/srv/ftp/multimedia` (lectura + escriptura allà).
- **`ftp3`** només tingui **lectura** a tot arreu.

```bash
sudo chown ftp1:ftp /srv/ftp/projectes
sudo chown ftp2:ftp /srv/ftp/multimedia
```

Perquè els usuaris engabiats hi puguin arribar, es fa amb un **enllaç simbòlic** dins de cada home (o bé s'ajusta el `chroot` — usem enllaços per simplicitat, si el `chroot` bloqueja l'accés a `/srv/ftp` cal treure l'engabiament d'aquest exercici o muntar-hi un `bind mount`).

Alternativa més pràctica per a la sessió: aquesta Part 3 es fa amb `ftp1` **fora de la gàbia** (recorda que és a la llista de `chroot_list`) i validem els permisos amb `ftp2` i `ftp3` connectant-se directament al seu home. Adapta l'exercici a la teva configuració.

Al **client**, crea uns fitxers de prova:

```bash
touch fitxer1.txt fitxer2.txt fitxer3.txt fitxer4.txt
```

### Fase A — Connexió i navegació

Connecta't com a `ftp1` primer per CLI (`ftp <IP>`) i després amb FileZilla:

1. Comprova el directori actual: `pwd`.
2. Llista el contingut remot: `ls`.
3. Llista el contingut local (des de dins del client FTP): `!ls`.
4. Canvia al `/srv/ftp/projectes` (o al directori equivalent al teu setup).

### Fase B — Pujar fitxers

Sempre primer CLI i després FileZilla:

1. Puja un fitxer sol: `put fitxer1.txt`.
2. Puja diversos a la vegada: `mput *.txt`.
3. Comprova amb `ls` que hi són.
4. Verifica amb FileZilla que veus el mateix.

### Fase C — Manipular directoris i fitxers

1. Crea un subdirectori: `mkdir informes`.
2. Canvia el nom d'un fitxer: `rename fitxer1.txt fitxer_renombrat.txt`.
3. Esborra un fitxer: `delete fitxer2.txt`.
4. Esborra el subdirectori (ha d'estar buit): `rmdir informes`.
5. Comprova cada operació amb FileZilla.

### Fase D — Descàrrega múltiple

1. Canvia el directori local del client: `lcd /tmp`.
2. Descarrega tots els `.txt` remots: `mget *.txt`.
3. Sortir amb `bye`.
4. Confirma a `/tmp` que hi ha arribat tot.

### Fase E — Comprovació de permisos (documentació obligatòria)

Per a cada acció, connecta't amb l'usuari indicat i **documenta el resultat esperat i el missatge que apareix**. Fes-ho **tant per CLI com per FileZilla**.

| Usuari | Acció | Directori | Resultat esperat |
|---|---|---|---|
| `ftp1` | `put fitxer.txt` | `/srv/ftp/projectes` | OK |
| `ftp1` | `put fitxer.txt` | `/srv/ftp/multimedia` | Denegat (`550`) |
| `ftp2` | `put fitxer.txt` | `/srv/ftp/multimedia` | OK |
| `ftp2` | `mkdir prova` | `/srv/ftp/projectes` | Denegat |
| `ftp3` | `get fitxer.txt` | qualsevol | OK |
| `ftp3` | `put fitxer.txt` | qualsevol | Denegat |
| `ftp3` | `delete fitxer.txt` | qualsevol | Denegat |
| anònim | `get fitxer.txt` | `/var/ftp/pub` | OK |
| anònim | `put fitxer.txt` | `/var/ftp/pub` | Denegat |

Recull els missatges d'error exactes (per exemple, `550 Permission denied`, `553 Could not create file`) — són material d'examen.

### Fase F — Repetir amb SFTP

Repeteix la Fase B, C i D amb **`sftp`** (usuari `ftp1`, port 22) i amb **FileZilla via SFTP**. Compara els missatges i observa que la sintaxi és pràcticament idèntica.

### Resultats esperats

- Totes les fases funcionen per CLI **i** amb FileZilla.
- Els permisos es respecten segons la taula i tens documentats els missatges d'error.
- SFTP dona una experiència pràcticament idèntica a FTP quan operes des del client — la diferència és a sota (SSH vs FTP).
