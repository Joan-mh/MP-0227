# Serveis de xarxa — Servidors proxy

## Què és un proxy

Un **servidor proxy** és un intermediari entre un client i un servidor. Actua com un "porter" pel qual passa tot el trànsit: en pot bloquejar una part, guardar-ne còpies (caché), portar-ne un registre i, en general, aplicar-hi qualsevol política que decideixi l'administrador.

Els beneficis tradicionals d'un proxy són:

- **Filtratge de contingut**: bloquejar accés a determinats llocs, protocols o horaris.
- **Cau (*cache*)**: quan un client demana una web, el proxy la baixa **una vegada** i la guarda. Les peticions posteriors al mateix contingut es responen des del cau, sense tornar a sortir a Internet. Estalvi d'ample de banda i millora de temps de resposta.
- **Registre (*logging*)**: cada connexió queda apuntada. Molt útil en xarxes corporatives i educatives per auditoria i estadístiques d'ús.
- **Anonimat parcial**: el proxy amaga la IP del client davant del servidor extern. La IP que veu el servidor és la del proxy, no la del client.
- **Protecció**: els clients no parlen directament amb Internet; el proxy fa de barrera.

## Proxy directe i proxy invers

Aquest és el punt conceptual clau del tema. Segons on es posi el proxy dins de la conversa, parlem d'un tipus o d'un altre.

### Proxy directe (*forward proxy*)

El proxy es col·loca **al costat dels clients**. Els clients estan configurats per fer-hi passar totes les seves peticions abans que arribin a Internet.

```
[CLIENT1]  ┐
[CLIENT2]  ├──►  [PROXY DIRECTE]  ──►  Internet  (servidors externs)
[CLIENT3]  ┘
```

Casos d'ús típics:

- **Xarxes corporatives i educatives** amb polítiques d'ús d'Internet.
- **Cauin de contingut** compartit entre molts usuaris.
- **Auditoria** de la navegació.

L'exemple clàssic és **Squid**.

### Proxy invers (*reverse proxy*)

El proxy es col·loca **al costat dels servidors**. Els clients d'Internet no parlen directament amb els servidors reals; parlen sempre amb el proxy, i aquest reparteix les peticions cap als servidors interns adequats.

```
Internet (clients) ──►  [PROXY INVERS]  ┬──►  [SERVIDOR APP 1]
                                        ├──►  [SERVIDOR APP 2]
                                        └──►  [SERVIDOR ESTÀTIC]
```

Casos d'ús típics:

- **Terminació TLS**: el proxy invers desxifra HTTPS un cop i envia HTTP net cap als servidors interns.
- **Balanceig de càrrega**: distribuir peticions entre diversos servidors idèntics per aguantar més usuaris.
- **Encaminament per ruta**: `/api/*` va a un servidor Node, `/static/*` a un altre, `/wp/*` a un tercer.
- **Cau i acceleració**: guardar contingut estàtic a prop del client (CDN).
- **Ocultació d'infraestructura**: el client no sap res dels servidors interns; només veu el proxy.

Les grans plataformes (Google, Netflix, Amazon...) tenen milers de proxies inversos distribuïts geogràficament (*edge servers*) per servir el contingut estàtic com més a prop possible dels usuaris finals.

L'exemple clàssic és **Nginx**.

### Resum ràpid

| Aspecte | Proxy directe | Proxy invers |
|---|---|---|
| Ubicació | Costat dels clients | Costat dels servidors |
| Qui el configura | Cada client (o el navegador) | Ningú, és transparent |
| Coneix el client? | Sí, és el seu (interno) | Sí, pero és Internet obert |
| Coneix el servidor destí? | El client decideix | El proxy decideix (per regles) |
| Cas d'ús típic | Filtratge/cau intern d'empresa | Balanceig, TLS, entrega ràpida |
| Exemple | Squid | Nginx |

## Squid

[Squid](https://www.squid-cache.org/) és el servidor proxy directe més utilitzat des dels anys 90. Codi lliure, llicència GPL, disponible als repositoris estàndard d'Ubuntu.

### Instal·lació i estat del servei

```bash
sudo apt update
sudo apt install squid
```

Comprovació de l'estat del servei:

```bash
sudo systemctl status squid
sudo systemctl restart squid
sudo systemctl reload squid    # més ràpid que restart; s'usa quan tocaste una ACL
```

### Fitxer de configuració

El fitxer principal és `/etc/squid/squid.conf`. Té **més de 9000 línies**, però la major part són comentaris amb exemples i documentació. Consells de supervivència amb `nano`:

- `Ctrl+C` — mostra a quina línia està el cursor.
- `Ctrl+W` — buscar text.
- `Ctrl+_` — anar directament a una línia.
- Obrir directament a una línia: `sudo nano +6000 /etc/squid/squid.conf`.

**Recomanació**: sempre fer còpia de seguretat abans de modificar-lo:

```bash
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
```

I validar la sintaxi abans de reiniciar:

```bash
sudo squid -k parse
```

### Directives essencials

- `http_port 3128` — port on escolta el proxy (per defecte 3128).
- `acl <nom> <tipus> <valor>` — defineix una llista d'accés amb un nom, un tipus i un valor.
- `http_access allow|deny <acl>` — aplica una ACL.
- `cache_dir ufs /var/spool/squid <MB> <L1> <L2>` — directori de cau amb la mida en MB i la profunditat de subdirectoris.

Un cop definit `cache_dir`, cal inicialitzar l'estructura del cau amb:

```bash
sudo squid -z
```

### Access Control Lists (ACLs)

Les ACLs són el cor d'Squid. Cada ACL té tres parts:

1. **Nom** (l'invents tu — únic dins de la config).
2. **Tipus** (què estem mirant: IP origen, IP destí, domini, URL, hora del dia...).
3. **Valor** (el patró concret a comparar).

Els tipus més usats a la pràctica:

| Tipus | Compara amb... |
|---|---|
| `src` | IP origen (client) |
| `dst` | IP destí (servidor extern) |
| `dstdomain` | Domini destí (per exemple, `www.exemple.cat`) |
| `dstdom_regex` | Domini destí, amb expressió regular |
| `url_regex` | La URL sencera, amb expressió regular |
| `urlpath_regex` | Només la part del path, amb expressió regular |
| `time` | Franja horària (`M T W H F` = dl-dv, hh:mm-hh:mm) |
| `port` | Port destí |

Llista completa i actualitzada: [wiki.squid-cache.org/SquidFaq/SquidAcl](https://wiki.squid-cache.org/SquidFaq/SquidAcl).

Una ACL **només defineix la norma**; per activar-la cal aplicar-la amb `http_access`.

### Combinació de normes

Aquesta és la part més subtil d'Squid. Cal entendre com es combinen les normes.

#### Verticalment (una sobre l'altra)

Les normes es llegeixen d'una en una, de dalt a baix. **Quan una coincideix (positivament o negativament), s'aplica i ja no es llegeix cap més**.

Exemple 1:

```
acl localnet src 192.168.30.0/24
acl ubuntuhost src 192.168.30.10
http_access allow localnet
http_access deny ubuntuhost
```

En llenguatge natural: *"si véns de la localnet, deixa'l passar. Si no, mira si ets ubuntuhost i denega-li"*.

Resultat: `ubuntuhost` **sí que pot navegar** — pertany a la localnet, la primera norma li coincideix i les altres no es miren.

Per denegar-lo, hem d'invertir l'ordre:

```
http_access deny ubuntuhost
http_access allow localnet
```

Ara `ubuntuhost` toca la primera norma, li denega, i ja no es mira res més.

#### Horitzontalment (una al costat de l'altra)

Diverses ACLs a la mateixa línia signifiquen **AND** (totes s'han de complir):

```
http_access allow localnet !ubuntuhost
```

*"Permet accedir si véns de la localnet **i** no ets ubuntuhost"*. El signe `!` és negació.

Es poden posar tantes com calgui:

```
http_access allow localnet !ubuntuhost horari_classe
```

### La regla implícita `deny all`

Sempre que es definesin ACLs personalitzades, cal **acabar el bloc amb un `deny all` explícit** perquè les peticions que no encaixen amb cap norma quedin denegades:

```
http_access allow localnet
http_access deny all
```

Sense `deny all`, el comportament pot ser imprevisible.

### Exercicis d'exemple resolts

**Exercici 1**: Què fa aquesta configuració?

```
acl network172 src 172.16.5.0/24
http_access allow network172
```

*Resposta*: només poden sortir a Internet els clients de la xarxa `172.16.5.0/24`. La resta reben la resposta per defecte, que sense `deny all` explícit dependrà de la config, però generalment `deny`.

**Exercici 2**: Què fa aquesta configuració?

```
acl Cooking2 dstdomain www.gourmet-chef.com
http_access deny Cooking2
http_access allow all
```

*Resposta*: es denega l'accés al domini `www.gourmet-chef.com` a tothom. La resta d'Internet queda permesa per a tothom.

(A l'apartat de pràctiques resoldreu més exercicis d'aquest tipus.)

### Fitxers de registre

Squid registra totes les peticions a `/var/log/squid/access.log`:

```bash
sudo tail -f /var/log/squid/access.log
```

Cada línia porta la data, la IP del client, l'URL demanada i si el proxy l'ha servit del cau o de fora.

Els errors del servei van a `/var/log/squid/cache.log`.

## Nginx com a proxy invers

[Nginx](https://nginx.org/) és el servidor web/proxy invers més utilitzat avui dia. Ja el vam presentar breument a la teoria d'RA5 com a alternativa a Apache; aquí veurem el seu paper com a **proxy invers**.

### Instal·lació

```bash
sudo apt update
sudo apt install nginx
```

Un cop instal·lat, escolta al port **80** amb una pàgina "Welcome to nginx!" per defecte.

### Configuració d'un proxy invers mínim

Els fitxers de configuració viuen a `/etc/nginx/sites-available/` i s'activen amb enllaços simbòlics a `/etc/nginx/sites-enabled/`. Un bloc `server` mínim que fa de proxy invers cap a un servidor backend:

```nginx
server {
    listen 80;
    server_name proxy.exemple.local;

    location / {
        proxy_pass http://192.168.30.20;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- `listen 80` — escolta al port 80.
- `server_name` — a quin nom de domini respon.
- `location /` — per a tots els paths.
- `proxy_pass` — cap a on redirigim.
- `proxy_set_header` — passem informació útil al backend perquè sàpiga qui és el client real i quin domini demana.

Aplicar canvis:

```bash
sudo nginx -t                # valida la sintaxi
sudo systemctl reload nginx  # aplica sense caiguda de servei
```

### Diferents paths cap a diferents backends

Un proxy invers pot decidir cap a on va cada URL segons el path:

```nginx
server {
    listen 80;
    server_name empresa.exemple.local;

    location /api/ {
        proxy_pass http://192.168.30.20:3000/;
    }

    location /static/ {
        proxy_pass http://192.168.30.21/;
    }

    location / {
        proxy_pass http://192.168.30.22/;
    }
}
```

Cada bloc `location` captura un prefix i el reenvia al servidor corresponent. Aquest patró és el que fan Netflix, Amazon, GitHub, etc. per repartir el trànsit entre desenes d'aplicacions internes.

### On acaba Nginx a aquest mòdul

En aquesta RA veurem Nginx només en la seva funció de **proxy invers senzill**: un frontal cap a un únic backend Apache. La configuració de **balanceig de càrrega** (múltiples backends amb `upstream` i `weight`) es reserva pel **Projecte 1**, on la treballarem en un escenari més complet.

## Recursos

- [Squid — documentació oficial](https://wiki.squid-cache.org/)
- [Squid ACLs — tipus complets](https://wiki.squid-cache.org/SquidFaq/SquidAcl)
- [Nginx — proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)
- Diferència visual entre proxy directe i invers: [www.youtube.com/watch?v=4NB0NDtOwIQ](https://www.youtube.com/watch?v=4NB0NDtOwIQ)
