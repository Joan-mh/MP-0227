# Serveis de xarxa — DNS

## Introducció: història i necessitat del DNS

Als inicis d'ARPANET, la xarxa predecessora d'Internet, la resolució de noms d'ordinador no era automàtica. Tots els nodes feien servir un fitxer anomenat **`hosts.txt`**, mantingut de manera centralitzada per la Stanford Research Institute (SRI).

Aquest fitxer contenia una llista de totes les adreces IP existents i els noms associats. Cada ordinador descarregava periòdicament una còpia actualitzada del fitxer i l'utilitzava per traduir noms a adreces IP.

```
127.0.0.1       localhost
192.168.1.10    servidor1.exemple.local servidor1
192.168.1.20    impressora.exemple.local impressora
8.8.8.8         google-dns
```

Aquest sistema funcionava bé mentre la xarxa era petita, amb pocs centenars de màquines. A mesura que ARPANET creixia i es transformava en l'Internet que coneixem, aquest model manual va començar a mostrar greus limitacions:

- Cada nou ordinador o canvi d'adreça havia de ser actualitzat manualment al fitxer central.
- La distribució de noves versions era lenta i podia provocar incoherències.
- La mida del fitxer creixia constantment i resultava poc pràctica la seva gestió.

Per solucionar aquests problemes, el 1983 **Paul Mockapetris** va dissenyar el **Domain Name System (DNS)**. Aquest nou sistema era jeràrquic i distribuït: permetia dividir la responsabilitat entre diferents servidors i escalar fàcilment a milers, i després milions, de dominis. El DNS introduïa la idea de **zones d'autoritat**, on cada organització podia administrar la seva part de l'espai de noms sense dependre d'un fitxer central.

Des de la seva creació, el DNS ha anat evolucionant amb noves extensions i mecanismes de seguretat, però l'essència continua sent la mateixa: un sistema distribuït i escalable per traduir noms en adreces IP. Avui dia és una **peça clau d'Internet**, comparable en importància a protocols bàsics com TCP/IP. Sense DNS, Internet seria pràcticament inusable per a la majoria d'usuaris.

Mireu aquest vídeo introductori (01 — Introducció) abans de continuar:
[01-Introducció](https://drive.google.com/file/d/1gAg1jVG-KQyGTeERLY2fiYlhqKQRfKWA/view?usp=drive_link)

## Què és el DNS

El **Domain Name System (DNS)** és un dels serveis fonamentals d'Internet i de qualsevol xarxa moderna. La seva funció principal és **traduir noms fàcils d'entendre per a les persones en adreces IP numèriques** que utilitzen els ordinadors per comunicar-se entre ells.

### Sistema jeràrquic i distribuït

El DNS està dissenyat com un sistema **jeràrquic i distribuït**:

- **Jeràrquic**: els noms de domini segueixen una estructura en forma d'arbre (arrel, dominis de primer nivell, subdominis, etc.).
- **Distribuït**: la informació no es troba en un únic lloc, sinó repartida en milers de servidors arreu del món. Això permet escalar i garantir disponibilitat.

### Traducció de noms a IP i viceversa

Quan un usuari escriu una adreça com `www.exemple.com`, el DNS la converteix en una IP, per exemple `93.184.216.34`. Com que l'ordinador només sap llegir adreces IP, però no noms de domini, cal fer aquesta transformació perquè el navegador (o una altra aplicació) demani la informació al servidor correcte.

El procés també pot funcionar a la inversa (**resolució inversa**): obtenir el nom de domini a partir d'una IP.

### Importància del DNS

Sense DNS, els usuaris haurien de memoritzar i escriure adreces IP per a cada pàgina web, impressora de xarxa o servei — cosa pràcticament impossible en una xarxa de gran mida. El DNS actua, doncs, com una **"guia telefònica d'Internet"**, que permet navegar i utilitzar serveis de manera intuïtiva i senzilla.

## Components bàsics del DNS

El funcionament del DNS es basa en diversos conceptes i rols que treballen conjuntament per resoldre noms de domini de manera ràpida i fiable.

### Dominis i subdominis

Un **domini** és un nom únic dins del sistema DNS, com ara `exemple.com`. Els **subdominis** permeten dividir aquest domini en parts més específiques, per exemple `mail.exemple.com` o `www.exemple.com`. Aquesta jerarquia en forma d'arbre facilita l'organització i l'administració de grans espais de noms.

### Zones d'autoritat

Una **zona d'autoritat** és la part de l'espai de noms DNS per la qual un servidor concret té responsabilitat. Per exemple, els servidors autoritatius del domini `exemple.com` contenen els registres que defineixen els subdominis i serveis d'aquest domini. Les zones permeten repartir la gestió del DNS entre diferents organitzacions o entitats, evitant la dependència d'un sol servidor central.

### Servidors DNS

Hi ha diferents tipus de servidors que intervenen en el procés de resolució:

- **Autoritatius**: contenen la informació definitiva d'una zona (els registres oficials). Si volem saber l'adreça de `www.exemple.com`, caldrà consultar el servidor autoritatiu de `exemple.com`.
- **Recursius**: actuen com a intermediaris. Reben la consulta d'un client i, si no tenen la resposta, s'encarreguen de fer totes les consultes necessàries fins a trobar-la. Els servidors recursius acostumen a ser els que ofereixen els proveïdors d'Internet (ISP) o serveis públics com Google DNS o Cloudflare.
- **De caché**: guarden temporalment respostes de consultes anteriors per accelerar resolucions futures. Aquesta memòria cau evita haver de repetir tot el procés cada vegada.

### Clients o resolvers

El **resolver** és la part del sistema que fa la consulta DNS en nom de l'usuari o del programa que la necessita. Quan escrivim una adreça al navegador, el nostre ordinador (a través del sistema operatiu) actua com a client DNS i envia la consulta al servidor recursiu configurat.

Vídeo recomanat (02 — Definicions):
[02-Definicions](https://drive.google.com/file/d/1rg7IqbeKTAf7XgjO1kKi6T40gC7HkOjF/view?usp=drive_link)

## Tipus de servidors DNS a la jerarquia

El sistema DNS és jeràrquic i depèn de diferents tipus de servidors que col·laboren entre si per resoldre una consulta.

### Root servers

Els **root servers** són el punt de partida del sistema DNS. N'hi ha 13 conjunts oficials, identificats per les lletres de la A fins a la M (per exemple, `a.root-servers.net`).

Tot i que només es parla de 13, en realitat hi ha centenars de còpies distribuïdes per **tot el món** gràcies a la tècnica **anycast**. Això assegura redundància, alta disponibilitat i velocitat de resposta.

Els root servers no contenen la informació de tots els dominis: **redirigeixen** a quins servidors TLD cal preguntar.

### TLD servers

Els **TLD (Top Level Domain) servers** gestionen els dominis de primer nivell: `.com`, `.org`, `.net`, `.es`, `.cat`, etc.

Quan un root server rep una consulta, respon indicant quin TLD server cal consultar. Per exemple, si es vol resoldre `www.exemple.com`, el root server indicarà el servidor de `.com`.

### Servidors autoritatius

Un **servidor autoritatiu** és el que té la informació definitiva sobre un domini concret. Per exemple, els servidors autoritatius de `exemple.com` contenen els registres que defineixen quina IP correspon a `www.exemple.com`, quin és el servidor de correu (MX), etc. Quan es fa una consulta DNS, finalment és un servidor autoritatiu qui dona la resposta correcta i fiable.

### Servidors recursius

Els **servidors recursius** són els que utilitzen habitualment els usuaris finals. Reben la consulta del client (ordinador, mòbil, etc.) i s'encarreguen de resoldre-la, preguntant als root servers, TLD i autoritatius si cal.

Exemples habituals:

- Els servidors DNS proporcionats pels ISP (proveïdors d'Internet).
- Els DNS públics com Google DNS (`8.8.8.8`, `8.8.4.4`), Cloudflare (`1.1.1.1`) o OpenDNS (`208.67.222.222`).

A més de fer de recursius, aquests servidors guarden en **caché** les respostes durant un temps (TTL), cosa que accelera molt les consultes repetides.

## Tipus de registres DNS

Els registres DNS són les entrades que descriuen com s'ha de resoldre un nom de domini. Cada registre té un **tipus** que defineix la seva funció i informació associada. Aquests registres els definirem en un fitxer de text al nostre servidor, quan ens faci falta.

Vídeos recomanats abans de continuar (04 — Registres, 05 — Resolució inversa):

- [04-Registres](https://drive.google.com/file/d/1-5jq-TDJQC_ZmgLCeMAYvdlbCGI-JZ64/view?usp=drive_link)
- [05-Resolució inversa](https://drive.google.com/file/d/1KaWTO5RCHrtEcHlwSezYFP3QYTvY_D_9/view?usp=drive_link)

### Registre A

Associa un nom de domini a una adreça **IPv4**.

```
www.exemple.com. IN A 93.184.216.34
```

Cada dispositiu de la nostra xarxa tindrà un registre `A` associat.

### Registre AAAA

Igual que l'`A`, però amb adreces **IPv6**.

```
www.exemple.com. IN AAAA 2606:2800:220:1:248:1893:25c8:1946
```

### Registre CNAME

Defineix un **àlies** per a un altre domini.

```
ftp.exemple.com. IN CNAME www.exemple.com.
```

El farem servir quan un servidor doni més d'un servei. A l'exemple, el nostre servidor ofereix FTP i HTTP.

### Registre MX

Indica quin servidor gestiona el correu electrònic d'un domini. Inclou una **preferència** (nombre baix = més prioritari).

```
exemple.com. IN MX 10 mail.exemple.com.
```

Aquest registre el farem servir, evidentment, quan treballem el RA4 (correu electrònic).

### Registre NS

Defineix els **servidors autoritatius** per a un domini.

```
exemple.com. IN NS ns1.exemple.com.
exemple.com. IN NS ns2.exemple.com.
```

Cada servidor de noms del nostre DNS ha de tenir un registre `NS`.

### Registre PTR

Utilitzat en la **resolució inversa** (d'una IP a un nom).

```
34.216.184.93.in-addr.arpa. IN PTR www.exemple.com.
```

Igual que amb els registres `A`, cada dispositiu amb IP assignada hauria de tenir un registre `PTR` si volem resolució inversa completa.

### Registre TXT

Permet afegir informació addicional en forma de text, sovint usada per seguretat i validacions (SPF, DKIM, verificacions de propietat).

```
exemple.com. IN TXT "v=spf1 include:_spf.google.com ~all"
```

### Registre SOA

**Start Of Authority**. Cada zona en té exactament un i defineix els paràmetres administratius de la zona:

```
@ IN SOA ns1.exemple.com. admin.exemple.com. (
                          1         ; Serial
                          604800    ; Refresh
                          86400     ; Retry
                          2419200   ; Expire
                          604800 )  ; Negative Cache TTL
```

Camps del SOA:

- **Serial**: número que cal incrementar cada cop que es modifica la zona. Els servidors secundaris (slaves) el consulten per saber si han d'actualitzar la seva còpia.
- **Refresh**: cada quan (en segons) el slave consulta el serial del master.
- **Retry**: si el slave no pot contactar el master, cada quan reintenta.
- **Expire**: si passat aquest temps el slave no ha pogut refrescar, deixa de servir la zona.
- **Negative Cache TTL**: quant de temps cacheja el resolver una resposta negativa (NXDOMAIN).

## Funcionament d'una consulta DNS

El procés de resolució DNS permet convertir un nom de domini en una adreça IP. Pot ser **recursiu** o **iteratiu**, depenent de com treballin els servidors implicats.

Mireu aquest vídeo abans de continuar (03 — Funcionament):
[03-Funcionament](https://drive.google.com/file/d/1uTCDnBapaVmy1eTO-3pBCHR8b1bC8Xrt/view?usp=drive_link)

### Resolució recursiva

En una resolució recursiva, el servidor DNS recursiu s'encarrega de consultar pas a pas fins a obtenir la resposta final i la retorna al client. És el mètode més habitual, utilitzat pels servidors DNS dels proveïdors d'Internet (ISP) i serveis públics com Google DNS o Cloudflare.

### Resolució iterativa

En la resolució iterativa, el servidor consultat respon amb la millor informació que té. Si no sap la resposta final, indica quin altre servidor s'ha de consultar. El client és qui ha d'anar fent les diferents consultes pas a pas fins arribar al servidor autoritatiu.

### Exemple pas a pas: consulta de www.exemple.com

1. L'usuari escriu `www.exemple.com` al navegador. El sistema operatiu comprova primer la seva memòria cau local i el fitxer `/etc/hosts`.
2. Si no troba la resposta, envia la consulta al servidor DNS recursiu configurat (per ex., el del seu ISP).
3. El servidor recursiu, si no té la resposta en caché, pregunta a un root server.
4. El root server no coneix l'adreça de `www.exemple.com`, però respon indicant on es troba el TLD server de `.com`.
5. El servidor recursiu consulta el TLD server de `.com`, que respon amb l'adreça dels servidors autoritatius de `exemple.com`.
6. El servidor recursiu consulta el servidor autoritatiu de `exemple.com`, que finalment respon amb l'adreça IP de `www.exemple.com`.
7. El servidor recursiu retorna la resposta al client i la guarda en caché durant el temps definit pel TTL.
8. L'usuari ja es pot connectar al servidor corresponent a `www.exemple.com`.

### Importància del TTL i la caché

El **TTL (Time To Live)** indica quant de temps una resposta pot ser emmagatzemada en caché. Això permet accelerar les consultes posteriors i reduir el trànsit a la xarxa. Quan el TTL expira, el servidor ha de tornar a consultar la informació.

## Fitxers relacionats a Ubuntu

### `/etc/resolv.conf`

Fitxer que indica quins servidors DNS utilitza el sistema per resoldre noms:

```
nameserver 192.168.50.1
```

> **Important**: aquest fitxer **no s'ha d'editar mai directament**. A Ubuntu modern, el gestiona `systemd-resolved` com a enllaç simbòlic (`/run/systemd/resolve/stub-resolv.conf`). El valor real ve del Netplan o del DHCP.

### `/etc/hosts`

Fitxer local que permet definir associacions fixes entre noms i IPs. Té prioritat sobre el DNS, però només és pràctic per a casos puntuals o proves. Avui dia el seu ús és molt limitat.

> **En aquest mòdul no el farem servir**. Cap configuració que funcioni gràcies a `/etc/hosts` es considerarà vàlida — la resolució ha de venir del servidor DNS.

## Bind9: servidor DNS a Linux

**`bind9`** (Berkeley Internet Name Domain, versió 9) és el programari més utilitzat a Linux per desplegar un servidor DNS complet. Pot actuar com a **servidor autoritatiu** o com a **servidor recursiu/caché**. És mantingut per **ISC** (Internet Systems Consortium).

### Instal·lació

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc
```

### Fitxers principals de configuració

Tots dins de `/etc/bind/`:

- **`named.conf`** — punt d'entrada. Inclou els altres fitxers (no es toca).
- **`named.conf.options`** — opcions globals: forwarders, listen-on, caché, DNSSEC.
- **`named.conf.local`** — declaració de les zones que gestionem (mestra, secundària, etc.).
- **`db.<zona>`** — fitxer de zona directa (registres `A`, `NS`, `MX`, `CNAME`, `TXT`...).
- **`db.<xarxa-inversa>`** — fitxer de zona inversa (registres `PTR`).

### Exemple bàsic complet

Suposem que volem servir el domini `exemple.local` a la xarxa `192.168.50.0/24`, amb `ns1.exemple.local = 192.168.50.1` i `www.exemple.local = 192.168.50.10`.

**1. Declaració de les zones a `/etc/bind/named.conf.local`:**

```
zone "exemple.local" {
    type master;
    file "/etc/bind/db.exemple.local";
};

zone "50.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.50";
};
```

> Recorda: la zona inversa d'una `/24` s'escriu **al revés**: `192.168.50.0/24` → `50.168.192.in-addr.arpa`.

**2. Opcions globals i forwarders a `/etc/bind/named.conf.options`:**

```
options {
    directory "/var/cache/bind";

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    // dnssec-validation auto;   // descomenta si vols validació DNSSEC

    auth-nxdomain no;    # RFC1035
    listen-on-v6 { any; };
};
```

Els **forwarders** són els servidors DNS externs als quals bind9 preguntarà si li arriba una consulta sobre un domini del qual **no és autoritatiu** i no té la resposta en caché.

**3. Zona directa `/etc/bind/db.exemple.local`:**

```
$TTL    604800
@       IN      SOA     ns1.exemple.local. admin.exemple.local. (
                              1         ; Serial
                              604800    ; Refresh
                              86400     ; Retry
                              2419200   ; Expire
                              604800 )  ; Negative Cache TTL
        IN      NS      ns1.exemple.local.
ns1     IN      A       192.168.50.1
www     IN      A       192.168.50.10
```

**4. Zona inversa `/etc/bind/db.192.168.50`:**

```
$TTL    604800
@       IN      SOA     ns1.exemple.local. admin.exemple.local. (
                              1         ; Serial
                              604800    ; Refresh
                              86400     ; Retry
                              2419200   ; Expire
                              604800 )  ; Negative Cache TTL
@       IN      NS      ns1.exemple.local.
1       IN      PTR     ns1.exemple.local.
10      IN      PTR     www.exemple.local.
```

**5. Reiniciar el servei i comprovar l'estat:**

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Amb aquesta configuració, qualsevol client que faci servir aquest servidor DNS podrà resoldre `www.exemple.local` a la IP `192.168.50.10` i la IP `192.168.50.10` a `www.exemple.local`.

### Detalls importants sobre la sintaxi dels fitxers de zona

- **El punt final és obligatori** en noms totalment qualificats (`ns1.exemple.local.`). Sense el punt, bind9 hi afegeix el nom de la zona: escriure `ns1.exemple.local` (sense punt) es converteix en `ns1.exemple.local.exemple.local.` — error habitual.
- El símbol `@` representa el nom de la zona actual.
- El **serial** del SOA s'ha d'incrementar **cada cop que es modifica la zona**. Convenció habitual: `AAAAMMDDNN` (any-mes-dia-versió).

### Eines de validació

Abans de reiniciar bind9, val la pena validar els fitxers:

```bash
sudo named-checkconf                                     # valida named.conf*
sudo named-checkzone exemple.local /etc/bind/db.exemple.local
sudo named-checkzone 50.168.192.in-addr.arpa /etc/bind/db.192.168.50
```

Un `OK` significa que la zona s'ha carregat correctament (mostra el serial detectat).

## Eines de diagnòstic: `dig` i `nslookup`

Aquestes són les úniques eines que farem servir per validar que el DNS funciona. **No fem servir `ping`** per comprovar el servei DNS — `ping` pot fallar per moltes causes que no tenen res a veure amb la resolució de noms.

### `dig`

Mostra la resposta completa de la consulta, capçaleres incloses. Molt útil per depurar.

```bash
dig www.google.com                        # consulta al DNS configurat al sistema
dig @192.168.50.1 www.exemple.local       # consulta un servidor concret
dig -x 192.168.50.10                      # resolució inversa
dig www.exemple.local +short              # només la resposta neta
```

### `nslookup`

Més senzill i universal (existeix també a Windows). Ideal per comprovacions ràpides:

```bash
nslookup www.exemple.local
nslookup www.exemple.local 192.168.50.1     # forçar un servidor concret
```

A Windows:

```
nslookup www.exemple.local
```

## Zones inverses

La zona inversa serveix per resoldre **d'IP a nom**. S'expressa amb el domini especial `in-addr.arpa` (per IPv4) o `ip6.arpa` (per IPv6). El nom de la zona s'escriu invertint els octets de la xarxa.

Exemples IPv4:

| Xarxa | Nom de zona inversa |
|---|---|
| `192.168.50.0/24` | `50.168.192.in-addr.arpa` |
| `10.0.0.0/8` | `10.in-addr.arpa` |
| `172.16.0.0/16` | `16.172.in-addr.arpa` |

Dins de la zona, cada host s'identifica només pel seu **últim octet** (o els octets que quedin fora de la part de xarxa):

```
1       IN      PTR     ns1.exemple.local.
10      IN      PTR     www.exemple.local.
```

Un mateix nom pot tenir molts `A` (per exemple, per repartir càrrega), però un `PTR` normalment només apunta a un nom canònic. Per això la resolució directa i la inversa **no sempre són simètriques** a la vida real.

### Nota sobre IPv6

Per a IPv6 el domini especial és `ip6.arpa` i la IP s'escriu **nibble a nibble** (grup de 4 bits) al revés. És més verbós, però la lògica és la mateixa. Es cobreix com a extra a la Part 2 de les pràctiques.

## Servidors secundaris (slaves) i transferència de zona

Un **servidor secundari** (històricament *slave*, avui també *secondary*) manté una còpia sincronitzada d'una zona servida per un altre servidor (*master*/*primary*). Serveix per:

- **Redundància**: si el master cau, el secundari continua responent.
- **Repartir càrrega** entre diversos servidors.

### Transferència de zona

Quan el secundari s'engega o quan el `Refresh` del SOA s'ha exhaurit, consulta el serial del master. Si el del master és superior, demana una **transferència de zona**:

- **AXFR**: transferència completa (còpia sencera de la zona).
- **IXFR**: transferència incremental (només els canvis).

Al master s'ha d'autoritzar explícitament la transferència amb la directiva `allow-transfer` a la zona corresponent. Sense això, per motius de seguretat, bind9 rebutja les peticions.

Al secundari, la zona es declara amb `type slave` (o `type secondary`) i s'indica la IP del master amb `masters { ... };`. Els fitxers transferits queden a `/var/cache/bind/` per defecte.

## Bones pràctiques i TLDs reservats

Encara que als exercicis podem inventar qualsevol domini, a la vida real cal evitar fer servir TLDs que siguin propietat d'una organització (`.edu`, `.com`, `.cat`, etc.) per a proves internes: podria causar col·lisions o confondre resolvers. Els TLDs que **la IANA reserva expressament per a proves i entorns privats** són (RFC 2606 i RFC 6761):

- `.test` — reservat per a proves.
- `.example` — reservat per a documentació i exemples.
- `.invalid` — reservat per garantir que la resolució sempre falli.
- `.localhost` — sempre s'ha de resoldre al loopback.
- `.local` — reservat de facto per xarxes locals (usat per mDNS/Bonjour; **compte** si convius amb aquests protocols).

Als exercicis d'aquest mòdul farem servir dominis del tipus `cognom.edu` (l'alumne substitueix pel seu cognom), però qui vulgui pot triar qualsevol dels TLDs de la llista anterior.

## Documentació i referències

- Guia clàssica de la comunitat Ubuntu: [help.ubuntu.com/community/BIND9ServerHowto](https://help.ubuntu.com/community/BIND9ServerHowto) — pot estar desactualitzada (última revisió important el 2018), però conserva bones explicacions conceptuals.
- Documentació oficial d'ISC (autoritat de bind9): [bind9.readthedocs.io](https://bind9.readthedocs.io/).
- RFCs fundacionals: **RFC 1034** i **RFC 1035** (definició del DNS).
