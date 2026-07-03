# Serveis de xarxa — HTTP (Apache)

## Introducció: què és HTTP

**HTTP** (*HyperText Transfer Protocol*) és el protocol de transferència de la web. És un protocol de la **capa d'aplicació** que segueix el model **client-servidor**: un client (habitualment un navegador) fa una **petició** a un servidor i aquest li retorna una **resposta** amb el contingut sol·licitat.

Punts clau:

- És un protocol **sense estat** (*stateless*): cada petició és independent, el servidor no recorda les anteriors. La sessió (login, carret de la compra...) es simula amb **cookies**.
- Utilitza per defecte el **port 80**.
- La versió original (HTTP/1.0) és del **1996**, HTTP/1.1 del 1997, **HTTP/2** del 2015 i **HTTP/3** del 2022. Avui la majoria de servidors admeten 1.1 i 2 alhora.
- És un protocol de **text pla** — el podem experimentar amb `telnet` o `curl -v`.

Mireu aquest vídeo introductori:
[Funcionament del protocol HTTP](https://drive.google.com/file/d/1CO42hU3SjmlgNtv38xMY8bmNZBCOMNSH/view?usp=share_link)

### HTTPS

**HTTPS** és HTTP dins d'una connexió xifrada amb **TLS** (l'evolució moderna de SSL). Utilitza per defecte el **port 443**. Un cop establerta la sessió TLS, els bytes que van amunt i avall són exactament HTTP normal — TLS només afegeix xifratge i verificació d'identitat (via certificat X.509).

Vídeo:
[Funcionament del protocol HTTPS](https://drive.google.com/file/d/19GmTzgcrd-mTGR9h7Blq4ZRMEHs67-Pt/view?usp=share_link)

**Important**: no confongueu **HTTP** (protocol) amb **HTML** (llenguatge). HTTP transporta qualsevol tipus de contingut (HTML, imatges, JSON, vídeo, PDF...) — HTML és només un d'aquests continguts.

## Servidors web: Apache vs. Nginx vs. IIS

Els tres principals servidors web que trobareu a Internet:

| Servidor | Origen | Fort en | Feble en |
|---|---|---|---|
| **Apache** | 1995, Apache Foundation. Codi obert. | Configuració modular, `.htaccess`, PHP amb `mod_php`, popularitat i documentació. | Rendiment amb moltes connexions simultànies (procés/fil per connexió). |
| **Nginx** | 2004, Igor Sysoev. Codi obert. Nom es llegeix "engine X". | Rendiment amb moltíssimes connexions (arquitectura d'esdeveniments), servei de contingut estàtic, proxy invers. | Sense equivalent tan simple a `.htaccess`; PHP obligat a passar per PHP-FPM. |
| **IIS** | Microsoft. Tancat, només Windows Server. | Integració amb l'ecosistema Microsoft (.NET, Active Directory). | No multiplataforma. |

Nginx va ser creat per resoldre el **problema C10K** ([wiki](https://es.wikipedia.org/wiki/Problema_C10k)): com atendre 10.000 connexions simultànies. Apache tradicionalment obre un procés o un fil per connexió; amb 10.000 clients això és inviable. Nginx atén milers de connexions amb un únic procés gràcies a un model orientat a esdeveniments.

Per això és molt habitual el patró **Nginx al davant + Apache al darrere** (proxy invers). Aquest patró es cobreix al **RA8 (Proxy)**, no aquí.

En aquest RA:

- **Pràctiques**: Apache com a servidor web principal + Nginx només com a comparativa (Part 4).
- **Projecte 1**: el servidor web pot ser Nginx (u altre) segons decisió del projecte.

## Apache: estructura del servidor

### Instal·lació

```bash
sudo apt update
sudo apt install apache2
apache2 -v
```

Es pot comprovar amb un navegador (o `lynx localhost`, `curl localhost`) que ja mostra la pàgina per defecte "Apache2 Default Page".

### Gestió del servei

Ordres bàsiques amb `systemctl`:

```bash
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2
sudo systemctl reload apache2      # recarrega config sense tallar sessions
sudo systemctl status apache2
```

També hi ha l'ordre històrica `apachectl`:

```bash
sudo apachectl restart
sudo apachectl graceful            # equivalent a reload, no desconnecta
sudo apachectl configtest          # valida sintaxi ABANS de reiniciar
```

**Consell important**: sempre `configtest` abans de reiniciar. Si hi ha un error de sintaxi, Apache no arrenca i tot el servei queda caigut.

### Estructura de fitxers a `/etc/apache2/`

```
/etc/apache2/
├── apache2.conf              # fitxer principal (inclou els altres)
├── envvars                   # variables d'entorn (usuari, grup)
├── ports.conf                # ports on escolta (Listen 80)
├── magic                     # signatures MIME
├── conf-available/           # fragments de configuració disponibles
│   └── *.conf
├── conf-enabled/             # enllaços simbòlics als anteriors (actius)
├── mods-available/           # tots els mòduls disponibles
│   └── *.load / *.conf
├── mods-enabled/             # enllaços simbòlics dels mòduls actius
├── sites-available/          # tots els llocs virtuals disponibles
│   └── *.conf
└── sites-enabled/            # enllaços simbòlics dels llocs actius
```

La convenció Debian/Ubuntu: **tot es prepara a `*-available/`** i s'**activa** amb un enllaç simbòlic a `*-enabled/`. Hi ha ordres per fer-ho fàcilment:

| Ordre | Fa |
|---|---|
| `a2enmod <nom>` | Activa un mòdul (link a `mods-enabled/`). |
| `a2dismod <nom>` | Desactiva el mòdul. |
| `a2ensite <nom>` | Activa un lloc virtual. |
| `a2dissite <nom>` | Desactiva'l. |
| `a2enconf <nom>` | Activa un fragment de configuració. |
| `a2disconf <nom>` | Desactiva'l. |

Un cop activat cal `sudo systemctl reload apache2` (o `restart`).

### Directori de continguts

- **`/var/www/html/`** — arrel dels continguts servits pel lloc per defecte.
- Es pot canviar amb la directiva `DocumentRoot`.

## Mòduls d'Apache

Apache carrega funcionalitats com a **mòduls** independents. La convenció de nom és `mod_<nom>` (per exemple, `mod_rewrite`, `mod_ssl`).

Llistar els mòduls actius:

```bash
apache2ctl -M
```

### Mòduls essencials (que activarem o ja estan actius)

| Mòdul | Què fa |
|---|---|
| `dir` | Serveix un fitxer índex (`index.html`, `index.php`) quan es demana un directori. |
| `alias` | Permet mapejar URLs a directoris del disc (`Alias`, `ScriptAlias`). |
| `autoindex` | Genera un llistat automàtic si no hi ha `index.html`. |
| `mime` | Assigna el `Content-Type` correcte segons l'extensió del fitxer. |
| `deflate` | Comprimeix les respostes (gzip). |
| `userdir` | Serveix pàgines des de `/home/<usuari>/public_html/` accessibles com `/~usuari`. |
| `rewrite` | Reescriu URLs (per redireccions, URLs amigables). |
| `headers` | Manipula les capçaleres HTTP de resposta. |
| `ssl` | Suport HTTPS (TLS). |
| `http2` | Habilita HTTP/2 (necessita `ssl`). |
| `auth_basic` | Autenticació HTTP bàsica (usuari + contrasenya). |
| `authn_file` | Backend d'autenticació basat en fitxer (`.htpasswd`). |
| `authz_core` | Autoritzacions modernes (`Require valid-user`, `Require ip`). |
| `socache_shmcb` | Cache de sessió TLS a memòria compartida (rendiment HTTPS). |

Els mòduls oficials: [httpd.apache.org/docs/2.4/mod/](https://httpd.apache.org/docs/2.4/mod/).

### Activar/desactivar

```bash
sudo a2enmod ssl headers rewrite
sudo systemctl reload apache2
```

## Directives principals

Les **directives** són instruccions que s'escriuen als fitxers de configuració. Format:

```
NomDirectiva valor
```

### Globals (a `apache2.conf` o `ports.conf`)

| Directiva | Ús |
|---|---|
| `Listen 80` | Port on escolta Apache. Es poden posar diversos. |
| `ServerName cognom.edu` | Nom principal del servidor. |
| `ServerAdmin admin@cognom.edu` | Correu de contacte que Apache posa a les pàgines d'error. |
| `Timeout 60` | Segons abans de tallar una connexió inactiva. |
| `KeepAlive On` | Reutilitzar la mateixa connexió TCP per a diverses peticions. |
| `MaxKeepAliveRequests 100` | Nombre màxim de peticions en una connexió. |

### Per lloc virtual (dins `<VirtualHost>`)

| Directiva | Ús |
|---|---|
| `DocumentRoot /var/www/lloc1` | Arrel dels continguts d'aquest lloc. |
| `ServerName www.lloc1.edu` | Nom pel qual s'accedeix. |
| `ServerAlias lloc1.edu` | Àlies addicionals. |
| `DirectoryIndex index.html index.php` | Fitxer(s) que es serveixen per defecte. |
| `ErrorLog ${APACHE_LOG_DIR}/lloc1_error.log` | Fitxer de log d'errors. |
| `CustomLog ${APACHE_LOG_DIR}/lloc1_access.log combined` | Fitxer de log d'accessos. |

## Servidors virtuals (VirtualHosts)

Un servidor Apache pot allotjar **múltiples llocs web** en la mateixa màquina i la mateixa IP, distingint-los pel **nom** amb què el client hi accedeix (capçalera `Host:` de la petició HTTP).

Estructura habitual:

```
/var/www/
├── virt1.edu/
│   └── index.html
├── virt2.edu/
│   └── index.html
└── virt3.edu/
    └── index.html
```

I un fitxer per cada lloc a `/etc/apache2/sites-available/`:

```apache
<VirtualHost *:80>
    ServerName www.virt1.edu
    ServerAlias virt1.edu
    DocumentRoot /var/www/virt1.edu
    ErrorLog ${APACHE_LOG_DIR}/virt1_error.log
    CustomLog ${APACHE_LOG_DIR}/virt1_access.log combined
</VirtualHost>
```

S'activen amb `a2ensite virt1` i cal reiniciar Apache. Perquè els clients hi arribin, el **DNS ha de resoldre els noms** — no fem servir `/etc/hosts` en aquest mòdul (veure RA2).

## `userdir`: pàgines web personals per usuari

Amb el mòdul **`userdir`** cada usuari del sistema pot tenir la seva pàgina personal accessible com `http://servidor/~usuari/`. Els fitxers viuen a `~/public_html/`.

Activació:

```bash
sudo a2enmod userdir
sudo systemctl reload apache2
```

L'usuari només ha de crear:

```bash
mkdir ~/public_html
echo "Hola, sóc l'usuari X" > ~/public_html/index.html
chmod 755 ~/public_html
```

Per defecte, la pàgina és accessible via `http://servidor/~usuariX/`.

## Autenticació bàsica moderna

Apache pot protegir directoris amb un usuari+contrasenya. En Apache **2.4+** la forma **recomanada** és fer-ho **directament al fitxer de configuració del site** amb un bloc `<Directory>`, **no** amb `.htaccess`.

### Per què evitar `.htaccess`

`.htaccess` és el fitxer que Apache llegeix a cada petició dins d'un directori. Té dos inconvenients:

- **Rendiment**: Apache el consulta a **cada petició** per veure si hi ha canvis. `<Directory>` es llegeix un cop en carregar la configuració.
- **Seguretat**: `AllowOverride All` deixa que qui pugui escriure al directori canviï el comportament del servidor. Un vector d'atac clàssic.

`.htaccess` només val la pena quan **no controlem la configuració global** del servidor (per exemple, un hosting compartit). En un servidor propi, mai. En aquest RA farem servir el **mètode modern** — vosaltres controleu la configuració.

### Directives modernes (Apache 2.4)

- `AuthType Basic` — mecanisme (Basic o Digest).
- `AuthName "Zona restringida"` — text que apareix al popup del navegador.
- `AuthUserFile /etc/apache2/.htpasswd` — fitxer amb usuaris i contrasenyes.
- `Require valid-user` — qualsevol usuari del fitxer, autenticat. També hi ha `Require user joan maria`, `Require ip 192.168.50.0/24`, etc.

### El fitxer `.htpasswd`

Es genera amb l'eina **`htpasswd`** (paquet `apache2-utils`):

```bash
# Crear el fitxer i afegir el primer usuari
sudo htpasswd -c /etc/apache2/.htpasswd usuari1

# Afegir més usuaris (SENSE -c, si no destrueix el fitxer!)
sudo htpasswd /etc/apache2/.htpasswd usuari2
```

**Compte** amb `-c`: si el poses una segona vegada, esborra el fitxer i el torna a crear amb només l'usuari nou.

## HTTPS: certificats i configuració

Per servir HTTPS necessitem un **certificat X.509** i la seva **clau privada**. Dues opcions:

### Certificat autofirmat (el que farem)

Generat amb OpenSSL al mateix servidor. Tècnicament vàlid però **no signat per cap CA**: els navegadors avisaran d'"amenaça de seguretat" i cal acceptar-lo manualment.

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/apache-mercade.key \
    -out /etc/ssl/certs/apache-mercade.crt \
    -subj "/CN=www.mercade.edu"
sudo chmod 600 /etc/ssl/private/apache-mercade.key
```

### El certificat "snakeoil" d'Ubuntu

Ubuntu ja instal·la un certificat autofirmat de mostra amb el paquet `ssl-cert`:

- `/etc/ssl/certs/ssl-cert-snakeoil.pem`
- `/etc/ssl/private/ssl-cert-snakeoil.key`

És el que fa servir el site `default-ssl` quan l'actives. Funciona però és **poc didàctic**: no aprens a generar-ne un, no té el teu domini al CN, i és compartit amb qualsevol Ubuntu. Al laboratori generem el nostre — es veu clarament al navegador quin CN té.

### Let's Encrypt

Autoritat certificadora gratuïta i automatitzada. Emet certificats **de veritat signats**. **No la fem servir aquí** perquè necessita un domini públic real accessible des d'Internet (com al RA4).

## Redirecció HTTP → HTTPS

En un servidor modern normalment **tot el trànsit del port 80 es redirigeix al 443**. Es fa amb un VirtualHost del port 80 que només conté una redirecció:

```apache
<VirtualHost *:80>
    ServerName www.mercade.edu
    Redirect permanent / https://www.mercade.edu/
</VirtualHost>
```

Amb `Redirect permanent` s'envia una resposta **301** al client, indicant-li que canviï la URL a HTTPS per sempre (el navegador ho recorda).

Alternativa amb `mod_rewrite`:

```apache
RewriteEngine On
RewriteCond %{HTTPS} !=on
RewriteRule ^/?(.*) https://%{SERVER_NAME}/$1 [R=301,L]
```

## Capçaleres de seguretat i bones pràctiques

### `ServerTokens Prod` + `ServerSignature Off`

Per defecte Apache posa capçaleres del tipus `Server: Apache/2.4.52 (Ubuntu)` a cada resposta, i afegeix una signatura a les pàgines d'error. Això dona pistes a un atacant sobre la versió. A `/etc/apache2/conf-available/security.conf`:

```apache
ServerTokens Prod
ServerSignature Off
```

Ara la capçalera passa a ser només `Server: Apache`.

### HSTS (Strict-Transport-Security)

Un cop l'usuari ha entrat una vegada per HTTPS, HSTS obliga el navegador a **no acceptar mai més HTTP** per aquell domini (durant el temps indicat). Protegeix contra atacs de *downgrade*.

```apache
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

Cal `sudo a2enmod headers`.

### HTTP/2

Suporta multiplexació (moltes peticions dins una única connexió) i comprimeix les capçaleres. Millora la velocitat de càrrega.

Activació:

```bash
sudo a2enmod http2
```

Al VirtualHost HTTPS:

```apache
Protocols h2 http/1.1
```

HTTP/2 **només funciona amb HTTPS** als navegadors — el pla és inaccessible.

## Introducció a Nginx

Nginx, com Apache, és un servidor web capaç de servir contingut estàtic i actuar com a proxy invers, balancejador de càrrega, cache HTTP i intermediari per a fastcgi (PHP).

- Instal·lació: `sudo apt install nginx`
- Fitxer principal: `/etc/nginx/nginx.conf`
- Llocs disponibles/actius: `/etc/nginx/sites-available/` i `/etc/nginx/sites-enabled/`

### Diferències bàsiques respecte Apache

| Aspecte | Apache | Nginx |
|---|---|---|
| Fitxer principal | `apache2.conf` | `nginx.conf` |
| Blocs | `<VirtualHost>` | `server { ... }` |
| Fitxer per directori | `<Directory>` | `location { ... }` |
| Autenticació bàsica | `auth_basic` (mòdul) | `auth_basic` (directiva) |
| PHP | `mod_php` (dins Apache) | Ha de passar per PHP-FPM |
| `.htaccess` | Sí | No existeix |
| Ports | `Listen` a `ports.conf` | `listen` dins `server` |

**A pràctiques**: Nginx només com a servidor web pur (Part 4). El seu rol com a proxy invers i balancejador es cobreix al **RA8**.

## Integració amb PHP

**PHP** és el llenguatge dinàmic més estès als servidors web. Es pot integrar amb Apache de dues formes:

- **`mod_php` (o `libapache2-mod-php`)** — el PHP s'executa dins del mateix procés d'Apache. Més simple, més ràpid per llocs petits, però el PHP consumeix memòria d'Apache sempre.
- **PHP-FPM** — el PHP s'executa en un **pool de processos separats**. Apache/Nginx els parla via **FastCGI**. Millor rendiment i aïllament. És l'estàndard modern.

A pràctiques farem la instal·lació més senzilla (`libapache2-mod-php`) i un fitxer `phpinfo.php` per comprovar que Apache serveix PHP. Al Projecte 1 pot canviar l'aproximació.

## Referències

- Documentació oficial d'Apache 2.4: [httpd.apache.org/docs/2.4/](https://httpd.apache.org/docs/2.4/)
- Guia d'Ubuntu sobre Apache: [ubuntu.com/server/docs/web-servers-apache](https://ubuntu.com/server/docs/web-servers-apache)
- Documentació oficial de Nginx: [nginx.org/en/docs/](https://nginx.org/en/docs/)
- RFC 7230-7237 — HTTP/1.1.
- RFC 7540 — HTTP/2.
- RFC 9114 — HTTP/3.
- [Problema C10K](https://es.wikipedia.org/wiki/Problema_C10k) — l'origen de Nginx.
