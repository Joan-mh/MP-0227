# Pràctiques RA5 — HTTP (Apache)

> **Xarxa**: `192.168.X.0/24`, el tercer octet `X` el trieu vosaltres. El **domini** l'inventeu (per exemple `cognom.edu`) i tindreu **DNS propi** funcionant (RA2). Els VirtualHosts amb nom **necessiten DNS**: `/etc/hosts` **no és vàlid** en aquest mòdul.
>
> **Client**: totes les proves es fan de base per **línia de comandes** al client Linux (`curl`, `lynx`, `wget`), però podeu fer les comprovacions **també** en un navegador (Firefox/Chrome al client Linux desktop o Windows) si ho preferiu — no és obligatori.

## Part 1 — Apache bàsic (2 hores)

### Objectius

- Instal·lar Apache i verificar-ne el funcionament.
- Conèixer l'estructura de fitxers de configuració.
- Identificar els mòduls actius i entendre la seva gestió.
- Canviar el `DocumentRoot` per defecte i el fitxer d'índex.
- Instal·lar `libapache2-mod-php` i servir un `phpinfo.php`.

### Tasques

#### 1. Instal·lació

1. `sudo apt update`.
2. Instal·la `apache2` i `apache2-utils` (les eines auxiliars — `htpasswd`, `apachectl`).
3. Comprova amb `apache2 -v` la versió instal·lada.
4. Verifica amb `curl http://localhost` (des del propi servidor) que apareix la pàgina "Apache2 Default Page".
5. Des del client Linux, `curl http://IP_SERVIDOR`, i comprova que veus el mateix.

#### 2. Exploració de fitxers

Fes un breu resum al vostre dossier de què hi ha a:

- `/etc/apache2/apache2.conf`
- `/etc/apache2/ports.conf`
- `/etc/apache2/sites-available/` i `sites-enabled/`
- `/etc/apache2/mods-available/` i `mods-enabled/`

#### 3. Identificació de mòduls

Llista els mòduls actius del vostre sistema:

```bash
apache2ctl -M
```

De la taula de la teoria (mòduls essencials), identifica **quins d'aquests estan actius per defecte** al teu sistema i **quins no**. Anota'ls al dossier.

#### 4. Canvi de `DocumentRoot`

Volem que el lloc per defecte serveixi els fitxers de **`/var/www/rapida`** i que el fitxer d'índex es digui **`inici.html`**.

1. Crea el directori `/var/www/rapida` i posa-hi un fitxer `inici.html` amb un contingut clar (per exemple, `<h1>Benvingut a la web ràpida</h1>`).
2. Edita `/etc/apache2/sites-available/000-default.conf` i:
   - Canvia el `DocumentRoot` per `/var/www/rapida`.
   - Afegeix una directiva `DirectoryIndex inici.html`.
   - Afegeix un bloc `<Directory /var/www/rapida>` amb `Require all granted` (accés obert).
3. Fes `sudo apachectl configtest`.
4. `sudo systemctl reload apache2`.
5. Comprova amb `curl http://IP_SERVIDOR/` que veus el nou contingut.

#### 5. Instal·lació de PHP amb Apache

1. `sudo apt install libapache2-mod-php`.
2. Comprova que el mòdul PHP està actiu: `apache2ctl -M | grep php`.
3. Crea `/var/www/rapida/phpinfo.php` amb el contingut:

    ```php
    <?php phpinfo(); ?>
    ```

4. Des del client, `curl http://IP_SERVIDOR/phpinfo.php | head -50` (o obre-ho al navegador). Has de veure la pàgina d'informació de PHP.
5. Al dossier, apunta la **versió de PHP** que ha reportat.

> **Compte**: `phpinfo()` deixa veure paràmetres interns del servidor. Un cop provat, esborra el fitxer o restringeix-ne l'accés — mai deixis un `phpinfo.php` accessible en producció.

### Resultats esperats

- Apache serveix una pàgina bàsica per HTTP.
- Has identificat els mòduls actius al teu sistema.
- El `DocumentRoot` per defecte apunta a `/var/www/rapida` amb `inici.html` com a índex.
- Un `phpinfo.php` s'executa correctament i reporta la versió de PHP instal·lada.

---

## Part 2 — VirtualHosts, `userdir` i autenticació bàsica (3 hores)

### Objectius

- Configurar **3 VirtualHosts amb noms** en una mateixa IP.
- Activar `mod_userdir` per pàgines personals dels usuaris.
- Protegir un directori amb **autenticació HTTP bàsica** feta **amb el bloc `<Directory>` moderne** (no amb `.htaccess`).

### Preparació DNS

Al vostre servidor DNS (RA2), afegiu al fitxer de zona:

```
www.virt1        IN      A       192.168.X.10
www.virt2        IN      A       192.168.X.10
www.virt3        IN      A       192.168.X.10
```

**Incrementeu el serial**, valideu amb `named-checkzone` i reinicieu bind9.

Verifiqueu des del client:

```bash
dig www.virt1.cognom.edu
dig www.virt2.cognom.edu
dig www.virt3.cognom.edu
```

Tots tres han de resoldre a la IP del servidor.

### Tasques

#### 1. Creació de l'estructura de continguts

Al servidor:

```bash
sudo mkdir -p /var/www/virt1 /var/www/virt2 /var/www/virt3
echo "<h1>Aquest és el lloc VIRT1</h1>" | sudo tee /var/www/virt1/index.html
echo "<h1>Aquest és el lloc VIRT2</h1>" | sudo tee /var/www/virt2/index.html
echo "<h1>Aquest és el lloc VIRT3</h1>" | sudo tee /var/www/virt3/index.html
```

#### 2. VirtualHosts

Crea tres fitxers a `/etc/apache2/sites-available/`:

- `virt1.conf`, `virt2.conf`, `virt3.conf`.

Cadascun ha de tenir un bloc `<VirtualHost *:80>` amb:

- `ServerName www.virtX.cognom.edu` i `ServerAlias virtX.cognom.edu`.
- `DocumentRoot /var/www/virtX`.
- Un bloc `<Directory /var/www/virtX>` amb `Require all granted`.
- `ErrorLog` i `CustomLog` propis.

Activa'ls i reinicia:

```bash
sudo a2ensite virt1 virt2 virt3
sudo apachectl configtest
sudo systemctl reload apache2
```

#### 3. Comprovació dels VirtualHosts

Des del client, comprova que cada domini retorna el contingut correcte:

```bash
curl http://www.virt1.cognom.edu
curl http://www.virt2.cognom.edu
curl http://www.virt3.cognom.edu
```

També pots comprovar-ho al navegador (Linux desktop o Windows). Cada URL ha de mostrar el text del lloc corresponent.

#### 4. `userdir`: pàgines personals

Al servidor:

1. Activa el mòdul: `sudo a2enmod userdir` i recarrega Apache.
2. Crea dos usuaris del sistema: `http1` i `http2` (amb `adduser`, amb el shell per defecte).
3. Com a cada usuari, o des de root:

    ```bash
    sudo -u http1 mkdir /home/http1/public_html
    echo "<h1>Web de http1</h1>" | sudo -u http1 tee /home/http1/public_html/index.html
    sudo chmod 711 /home/http1
    sudo chmod 755 /home/http1/public_html
    ```

    Repeteix el mateix per `http2`.
4. Des del client, accedeix a `http://IP_SERVIDOR/~http1/` i `http://IP_SERVIDOR/~http2/`.

#### 5. Directori protegit amb autenticació bàsica (mètode **modern**, sense `.htaccess`)

Crearem un directori protegit dins `virt1` on **només poden entrar `http1` i `http2`**.

1. Crea el directori i el contingut:

    ```bash
    sudo mkdir /var/www/virt1/privat
    echo "<h1>Zona privada de virt1</h1>" | sudo tee /var/www/virt1/privat/index.html
    ```

2. Crea el fitxer de contrasenyes amb `htpasswd` (paquet `apache2-utils`):

    ```bash
    sudo htpasswd -c /etc/apache2/.htpasswd http1
    sudo htpasswd /etc/apache2/.htpasswd http2
    ```

    ⚠️ El **`-c` només la primera vegada** — la segona vegada esborra el fitxer.

3. Edita `/etc/apache2/sites-available/virt1.conf` i afegeix (dins del `<VirtualHost>` corresponent):

    ```apache
    <Directory /var/www/virt1/privat>
        AuthType Basic
        AuthName "Zona restringida de VIRT1"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>
    ```

4. Assegura't que els mòduls d'autenticació estan actius:

    ```bash
    sudo a2enmod auth_basic authn_file authz_user
    ```

5. `sudo apachectl configtest` i `sudo systemctl reload apache2`.

#### 6. Comprovació de l'autenticació

- `curl -I http://www.virt1.cognom.edu/privat/` ha de retornar `401 Unauthorized`.
- `curl -u http1 http://www.virt1.cognom.edu/privat/` (et demanarà la contrasenya) ha de retornar el contingut.
- Amb el navegador ha d'aparèixer un popup demanant usuari i contrasenya.

#### 7. Reflexió obligatòria al dossier

- Per què és millor `<Directory>` que `.htaccess`?
- Què li passaria a la vostra configuració si permetéssiu `AllowOverride All` a `/var/www/`?

### Resultats esperats

- Els 3 VirtualHosts responen amb el contingut correcte per DNS.
- Els usuaris `http1` i `http2` tenen web personal accessible per `/~usuari/`.
- El directori `/privat/` demana usuari i contrasenya i només els usuaris del `.htpasswd` hi poden entrar.

---

## Part 3 — HTTPS, redirecció, HTTP/2 i capçaleres de seguretat (2 hores)

### Objectius

- Servir un lloc per HTTPS amb un **certificat autofirmat propi** generat amb OpenSSL.
- Comparar-lo amb el certificat **`snakeoil`** que ve amb Ubuntu.
- Configurar la **redirecció HTTP → HTTPS**.
- Activar **HTTP/2**, **HSTS** i ocultar la versió d'Apache (**`ServerTokens Prod`**).

### Tasques

#### 1. Certificat autofirmat propi

Genera clau i certificat per al domini `www.virt1.cognom.edu`:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/virt1.key \
    -out /etc/ssl/certs/virt1.crt \
    -subj "/CN=www.virt1.cognom.edu" \
    -addext "subjectAltName=DNS:www.virt1.cognom.edu,DNS:virt1.cognom.edu"
sudo chmod 600 /etc/ssl/private/virt1.key
```

Verifica'l:

```bash
sudo openssl x509 -in /etc/ssl/certs/virt1.crt -noout -subject -dates
```

#### 2. VirtualHost HTTPS

Activa el mòdul SSL:

```bash
sudo a2enmod ssl
```

Crea el fitxer `/etc/apache2/sites-available/virt1-ssl.conf`:

```apache
<VirtualHost *:443>
    ServerName www.virt1.cognom.edu
    ServerAlias virt1.cognom.edu
    DocumentRoot /var/www/virt1

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/virt1.crt
    SSLCertificateKeyFile   /etc/ssl/private/virt1.key

    <Directory /var/www/virt1>
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/virt1_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/virt1_ssl_access.log combined
</VirtualHost>
```

Activa'l i recarrega:

```bash
sudo a2ensite virt1-ssl
sudo apachectl configtest
sudo systemctl reload apache2
```

Comprova amb:

```bash
curl -k https://www.virt1.cognom.edu/
openssl s_client -connect www.virt1.cognom.edu:443 -servername www.virt1.cognom.edu </dev/null 2>/dev/null | openssl x509 -noout -subject
```

Si obres al navegador et sortirà l'avís d'"amenaça de seguretat" (perquè és autofirmat). Accepta'l manualment.

#### 3. Comparativa amb el certificat `snakeoil`

Ubuntu ja porta un certificat autofirmat de mostra:

```bash
ls -l /etc/ssl/certs/ssl-cert-snakeoil.pem /etc/ssl/private/ssl-cert-snakeoil.key
sudo openssl x509 -in /etc/ssl/certs/ssl-cert-snakeoil.pem -noout -subject -dates
```

Activa el site que ja porta Apache i que fa servir aquest certificat:

```bash
sudo a2ensite default-ssl
sudo systemctl reload apache2
```

Comprova:

```bash
openssl s_client -connect IP_SERVIDOR:443 </dev/null 2>/dev/null | openssl x509 -noout -subject
```

**Al dossier**: compara els dos certificats. Què veus al `subject`? El nostre porta el nom del domini (`CN = www.virt1.cognom.edu`); el snakeoil porta un CN genèric del sistema. Per què el nostre és més útil?

Desactiva el `default-ssl` per no confondre les proves posteriors:

```bash
sudo a2dissite default-ssl
sudo systemctl reload apache2
```

#### 4. Redirecció HTTP → HTTPS

Edita `/etc/apache2/sites-available/virt1.conf` (el VirtualHost del port 80) i **substitueix el `DocumentRoot`** per una redirecció:

```apache
<VirtualHost *:80>
    ServerName www.virt1.cognom.edu
    ServerAlias virt1.cognom.edu
    Redirect permanent / https://www.virt1.cognom.edu/
</VirtualHost>
```

Recarrega i comprova:

```bash
curl -I http://www.virt1.cognom.edu/
```

Ha de retornar un `301 Moved Permanently` amb `Location: https://www.virt1.cognom.edu/`.

Amb `curl -L` seguirà la redirecció:

```bash
curl -kL http://www.virt1.cognom.edu/
```

#### 5. HTTP/2

```bash
sudo a2enmod http2
```

Al fitxer `virt1-ssl.conf`, dins el `<VirtualHost *:443>`, afegeix:

```apache
Protocols h2 http/1.1
```

Recarrega. Comprova amb:

```bash
curl -kIv --http2 https://www.virt1.cognom.edu/ 2>&1 | grep -E "HTTP/2|ALPN"
```

Has de veure `HTTP/2 200` (o similar) i `ALPN: server accepted h2`.

#### 6. HSTS

Activa el mòdul `headers`:

```bash
sudo a2enmod headers
```

Al `virt1-ssl.conf`, dins el `<VirtualHost *:443>`, afegeix:

```apache
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

Recarrega. Comprova:

```bash
curl -kI https://www.virt1.cognom.edu/ | grep -i strict-transport
```

Ha d'aparèixer la capçalera `Strict-Transport-Security`.

#### 7. Ocultar la versió d'Apache

Edita `/etc/apache2/conf-available/security.conf` i modifica:

```
ServerTokens Prod
ServerSignature Off
```

Recarrega. Comprova:

```bash
curl -I http://www.virt1.cognom.edu/ | grep -i server
```

Abans posava `Server: Apache/2.4.52 (Ubuntu)`. Ara ha de posar només `Server: Apache`.

### Entrega

- Captures o sortides de `curl` de cada pas (`-k` per acceptar autofirmats, `-I` per veure només capçaleres).
- Comparativa entre el certificat propi i el snakeoil.
- Nota breu explicant per què no fem servir Let's Encrypt en aquest RA.

### Resultats esperats

- HTTPS funcional amb el certificat propi.
- Tot el trànsit HTTP redirigit a HTTPS.
- HTTP/2 actiu.
- Capçalera HSTS present.
- La versió d'Apache no es filtra al banner `Server:`.

---

## Part 4 — Nginx com a servidor web (introducció) (1 hora)

> Nginx com a **proxy invers i balancejador** es fa a **RA8**. Aquí només veureu Nginx servint contingut estàtic i compararem la seva sintaxi amb la d'Apache.

### Objectius

- Instal·lar Nginx en una **segona VM** (no barrejar amb l'Apache anterior).
- Comparar l'estructura de fitxers amb Apache.
- Servir una pàgina estàtica bàsica.

### Preparació

Necessiteu una VM nova (o desactivar Apache a la primera per alliberar el port 80 — millor VM nova per no perdre la config d'Apache).

### Tasques

#### 1. Instal·lació

```bash
sudo apt install nginx
sudo systemctl status nginx
```

Amb `curl http://localhost` (o al navegador) ha d'aparèixer la pàgina "Welcome to nginx!".

#### 2. Explorar l'estructura

Fes un breu resum al dossier de què hi ha a:

- `/etc/nginx/nginx.conf`
- `/etc/nginx/sites-available/`
- `/etc/nginx/sites-enabled/`

#### 3. Comparativa de sintaxi (dossier)

Completa aquesta taula al dossier amb el fitxer equivalent o directiva equivalent per a cada cas:

| Concepte | Apache | Nginx |
|---|---|---|
| Fitxer principal | `apache2.conf` | ? |
| Directori de continguts per defecte | `/var/www/html` | ? |
| Bloc que defineix un lloc virtual | `<VirtualHost *:80>` | ? |
| Port on escolta | `Listen 80` a `ports.conf` | ? |
| Directori dins d'un lloc | `<Directory>` | ? |
| Ordre per comprovar sintaxi | `apachectl configtest` | ? |

#### 4. Servir una pàgina estàtica personalitzada

1. Crea `/var/www/rapida-nginx/index.html` amb un contingut diferenciat.
2. Edita `/etc/nginx/sites-available/default` i canvia el `root` per `/var/www/rapida-nginx`.
3. `sudo nginx -t` per validar la sintaxi.
4. `sudo systemctl reload nginx`.
5. Comprova amb `curl` que serveix la nova pàgina.

### Resultats esperats

- Nginx serveix una pàgina estàtica personalitzada.
- El dossier té la taula comparativa Apache/Nginx completa.

> Al **RA8** us endinsareu en el rol més potent de Nginx: **proxy invers** i **balancejador de càrrega**.
