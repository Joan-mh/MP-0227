# Pràctiques RA8 — Proxy

> **Escenari de xarxa** (Parts 1 i 2):
>
> - **Servidor Ubuntu (proxy)** amb **dues interfícies**:
>   - `enp0s3` — **NAT** (per baixar paquets i sortir a Internet)
>   - `enp0s8` — **xarxa interna** (o host-only), IP estàtica `192.168.30.1/24`
> - **Client Linux** i **Client Windows** (opcional) connectats a la mateixa xarxa interna, agafant IP dinàmica del rang de proves o estàtica dins de `192.168.30.0/24`.
> - Els clients es configuren manualment per fer servir el proxy al port `3128` (Part 1 i 2) o `80` (Part 3).
>
> **Client principal**: Ubuntu Desktop (curl, navegador Firefox). El client Windows és **opcional** per validar que el proxy és agnòstic al sistema del client.
>
> **Xarxa Part 3**: canvia per l'escenari de proxy invers — veure la Part 3.

## Part 1 — Squid: configuració bàsica i cau

### Objectius

- Instal·lar Squid al servidor Ubuntu.
- Definir una ACL per a la xarxa local de clients.
- Configurar un directori de cau i inicialitzar-lo.
- Verificar el funcionament des dels clients i llegir el fitxer de registre.

### Tasques

#### 1. Instal·lació i estat

1. Al servidor, actualitza els repositoris i instal·la Squid:

   ```bash
   sudo apt update
   sudo apt install squid
   ```

2. Comprova la versió i l'estat del servei:

   ```bash
   squid -v
   sudo systemctl status squid
   ```

3. Fes còpia de seguretat del fitxer de configuració:

   ```bash
   sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
   ```

#### 2. ACL per a la xarxa local

Localitza a `/etc/squid/squid.conf` la secció de definicions d'ACL (aprox. línia 1000; fes servir `Ctrl+W` per buscar `acl localnet`). Comenta totes les ACL `localnet` predefinides i afegeix una única línia amb la teva xarxa:

```
acl localnet src 192.168.30.0/24
```

#### 3. Activar `http_access` per la localnet

Al mateix fitxer, cerca la línia que posa `http_access allow localnet` (comentada per defecte). Descomenta-la i deixa **just abans** de `http_access deny all` un bloc final que quedi així:

```
http_access allow localnet
http_access deny all
```

#### 4. Comprovar el port d'escolta

Cerca `http_port 3128`. Verifica que està **sense comentar**. Deixa el port `3128` (per defecte).

#### 5. Configurar el cau

Cerca `cache_dir` i **descomenta** la línia:

```
cache_dir ufs /var/spool/squid 100 16 256
```

- `100` = MB de cau
- `16` i `256` = subdirectoris de nivell 1 i 2

#### 6. Validar la configuració i inicialitzar el cau

```bash
sudo squid -k parse
sudo squid -z
sudo systemctl restart squid
```

Si `squid -k parse` no dona errors, es pot reiniciar el servei.

#### 7. Configurar els clients

- **Client Linux (Firefox)**: **Configuració → General → Configuració de xarxa → Configuració manual del proxy** → *HTTP proxy* = IP del servidor Squid, *Port* = `3128`, marca "També fer servir aquest proxy per HTTPS".
- **Client Linux (curl)** — sense tocar el navegador:

  ```bash
  curl -x http://192.168.30.1:3128 http://neverssl.com
  ```

- **Client Windows (opcional)**: Panell de control → Opcions d'Internet → Connexions → Configuració LAN → *Servidor proxy* activat, IP i port `3128`.

#### 8. Verificar el registre

Al servidor, mira les peticions que passen pel proxy:

```bash
sudo tail -f /var/log/squid/access.log
```

Des dels clients, navega per unes quantes pàgines i observa les línies que apareixen al log en temps real.

### Verificació

- El servei `squid` està actiu i escolta al port `3128`.
- El client Linux (i opcionalment el Windows) navega **només** si té el proxy configurat; si el treiem, ja no li surt Internet a través de la xarxa interna.
- El fitxer `/var/log/squid/access.log` mostra les URLs demanades per cada client.

### Reflexió

- Per què és útil la cau d'un proxy en xarxes amb ample de banda limitat?
- Què passaria si el proxy caigués?

## Part 2 — Squid: ACLs avançades i restriccions

### Objectius

- Definir ACLs per bloquejar hosts, dominis, patrons d'URL i franges horàries.
- Entendre la combinació vertical i horitzontal de normes.
- Analitzar el registre per confirmar les restriccions aplicades.

### Consell d'organització

**Poseu totes les ACLs i tots els `http_access` en un mateix bloc al final del fitxer**, marcat amb un comentari `# --- ACLs personalitzades ---`. Amb 9000 línies, dispersar-los és una recepta per equivocar-se.

### Recordatori

Per cada URL de test:

```
http://server.example.com/some/path/to/an/object
```

- `url_regex` compara amb tota la URL.
- `urlpath_regex` compara només amb `/some/path/to/an/object`.
- `dstdom_regex` compara amb `server.example.com`.
- `dstdomain` compara amb el domini **exacte** (no expressió regular).

### Exercici previ (en paper)

**Abans de tocar el fitxer**, mira les tres configuracions següents i respon què fa cadascuna. Anota-ho al dossier per corregir després.

**Config A**:

```
acl ME src 10.0.0.1
acl YOU src 10.0.0.2
http_access allow ME YOU
```

**Config B**:

```
acl US src 10.0.0.1 10.0.0.2
http_access allow !US
http_access deny all
```

**Config C**:

```
acl USER1 src 192.168.100.0/24
acl USER2 src 192.168.200.0/24 192.168.201.0/24
acl DAY time 06:00-18:00
http_access allow USER1 DAY
http_access deny USER1
http_access allow USER2 !DAY
http_access deny USER2
```

### Tasques

Configureu Squid per satisfer les restriccions següents. Recordeu: cada canvi validat amb `sudo squid -k parse` i aplicat amb `sudo systemctl reload squid`.

#### 1. Bloqueig d'un host per **nom** (no per IP)

L'equip Windows (per exemple `windows.aula.local`) **no ha de tenir accés a Internet**. La resta d'equips de la xarxa local sí. Per bloquejar per nom, cal:

- Al DNS o al `/etc/hosts` del servidor Squid, tenir el nom resoluble.
- Fer servir el tipus `srcdomain` (o `dstdomain` segons el cas).

Escriu la configuració i prova-la.

#### 2. Llista d'URLs prohibides

Bloqueja aquests dominis a tota la xarxa:

- `www.gencat.cat`
- `www.fcbarcelona.cat` o `www.realmadrid.com` (tria un dels dos)
- `www.cirvianum.info`

Fes servir `dstdomain`.

#### 3. Llista de paraules prohibides a la URL

Amb `url_regex`, bloqueja les URLs que continguin **qualsevol** d'aquestes paraules:

- `Torello`
- `gamarus`
- `tartera`
- `bonsai`
- `facebook`
- `lol`

Consell: `url_regex` pot rebre múltiples patrons a la mateixa línia o pot llegir-los d'un fitxer:

```
acl paraules url_regex "/etc/squid/paraules-prohibides.txt"
```

#### 4. Bloqueig de descàrregues d'executables

Investiga i crea una ACL que eviti descarregar fitxers acabats en `.exe`, `.msi` i `.bat`. Fes servir `urlpath_regex`.

Pista: `\.exe$` és una expressió regular que coincideix amb URLs que acaben en `.exe`.

#### 5. Restricció horària

Crea una ACL per permetre l'ús del proxy **només durant l'horari lectiu** (per exemple, dilluns a divendres de 8:00 a 15:00) i denegar la resta d'hores. Fes proves canviant l'hora del sistema (`sudo timedatectl set-time '2026-07-04 14:00:00'`) per demostrar que funciona.

#### 6. La resta ha d'estar permès

Deixa `http_access allow localnet` al final abans del `deny all` per garantir que els clients no bloquejats per cap norma anterior poden navegar.

#### 7. Comprovacions al registre

- Fes proves des dels clients: intenta accedir a alguna URL bloquejada i alguna permesa.
- A `/var/log/squid/access.log` mira com apareixen les accions: `TCP_DENIED` per les bloquejades, `TCP_MISS` o `TCP_HIT` per les que passen.

### Verificació

- Els accessos bloquejats es rebutgen amb error de proxy al navegador.
- Al log s'hi veu clarament què s'ha denegat i què s'ha permès.
- La restricció horària deixa d'aplicar-se quan es canvia l'hora del sistema.

### Reflexió

- Quina diferència pràctica hi ha entre combinar normes verticalment i horitzontalment?
- Quins riscos té un `url_regex` mal escrit? (Pista: `cooking` també coincideix a `Cooking-Recipes.com` si no es marca com a *case-insensitive*.)

## Part 3 — Nginx com a proxy invers

### Escenari

Aquesta part canvia d'esquema: ara els clients són d'"Internet" i el proxy es col·loca **davant** dels servidors.

- **MV1 — Nginx (frontal)**: rep les peticions dels clients. Interfícies: NAT (per instal·lar) + xarxa interna al mateix rang que MV2.
- **MV2 — Apache (backend)**: serveix una pàgina web reconeixible.
- **Client**: qualsevol màquina que arribi a MV1 (fins i tot MV1 mateix, per curl).

Podeu reutilitzar la MV Apache de la RA5 com a backend.

### Objectius

- Instal·lar Nginx i verificar-ne el funcionament com a servidor web.
- Configurar-lo com a proxy invers cap a un servidor Apache backend.
- Comprovar que el client, quan demana la IP de Nginx, rep el contingut d'Apache.

### Tasques

#### 1. Preparar el backend Apache (MV2)

A la MV2, comprova que Apache està actiu:

```bash
sudo systemctl status apache2
curl http://localhost
```

Modifica el fitxer `/var/www/html/index.html` per posar un contingut clarament identificable:

```html
<h1>Sóc el servidor Apache backend</h1>
```

Anota la IP de MV2 (per exemple `192.168.30.20`).

#### 2. Instal·lar Nginx a MV1

```bash
sudo apt update
sudo apt install nginx
sudo systemctl status nginx
```

Verifica des d'un navegador (o `curl http://IP_MV1`) que apareix la pàgina de benvinguda de Nginx.

**Captura de pantalla** de la pàgina per defecte.

#### 3. Configurar Nginx com a proxy invers

Edita `/etc/nginx/sites-available/default` (o crea un nou fitxer `/etc/nginx/sites-available/proxy` i actíva'l amb un enllaç simbòlic a `sites-enabled/`).

Substitueix el bloc `location /` perquè quedi així:

```nginx
server {
    listen 80 default_server;
    server_name _;

    location / {
        proxy_pass http://192.168.30.20;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Substitueix `192.168.30.20` per la IP real de la teva MV2.

#### 4. Validar i aplicar

```bash
sudo nginx -t
sudo systemctl reload nginx
```

#### 5. Prova des del client

Des del client Linux, connecta't a la IP de **MV1** (no de MV2):

```bash
curl http://IP_MV1
```

Ha d'aparèixer el contingut d'Apache (`<h1>Sóc el servidor Apache backend</h1>`), servit a través de Nginx.

**Captura de pantalla** del navegador mostrant la IP de MV1 a la barra d'adreces i el contingut d'Apache al cos.

#### 6. Provar la ruta amb diferents `location`

Modifica la config perquè:

- Peticions a `/apache/` vagin al backend Apache.
- Peticions a `/` continuïn servint la pàgina per defecte de Nginx.

Pista: dins d'un bloc `server` pots tenir diversos blocs `location`, i has de treure `default_server` del `listen` o gestionar-ho amb `root`.

### Verificació

- Accedint a la IP de MV1, l'usuari veu contingut que **físicament resideix a MV2**.
- Els fitxers de log de Nginx (`/var/log/nginx/access.log`) mostren les peticions com si tots els clients d'"Internet" (fins i tot els d'una xarxa privada) arribessin al proxy.
- El fitxer de log d'Apache a MV2 mostra que les peticions arriben des de la IP de Nginx, no des dels clients reals — a menys que fem servir `X-Real-IP`, que aleshores queda al log.

### Reflexió

- Per què el balanceig de càrrega és un cas natural del proxy invers?
- Quina diferència pràctica hi ha entre un proxy directe i un proxy invers **des del punt de vista del client**?
- Per què les grans plataformes (Google, Netflix...) tenen milers de proxies inversos distribuïts pel món?
