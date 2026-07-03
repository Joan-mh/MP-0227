# Serveis de xarxa — Configuració de xarxa i DHCP

## Configuració de la xarxa a Ubuntu Linux

En un sistema Linux modern com **Ubuntu Server**, la configuració de xarxa és essencial per poder connectar-se a Internet o a una xarxa local. Aquesta configuració es pot fer:

- Manualment editant fitxers de configuració.
- Automàticament amb eines com NetworkManager o systemd-networkd.

Ubuntu, a partir de la versió 18.04, utilitza **Netplan** com a eina principal per gestionar la xarxa.

### Netplan i fitxers YAML

Netplan és un sistema que permet definir la configuració de xarxa en fitxers amb format YAML ubicats normalment a `/etc/netplan/`.

Un fitxer típic podria ser:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: true
```

Per canviar la configuració cal editar el fitxer amb el teu editor de text preferit.

**Explicació de l'exemple:**

- `version: 2` → versió de l'esquema de Netplan.
- `renderer: networkd` → indica quin sistema aplicarà la configuració (systemd-networkd o NetworkManager).
- `ethernets` → defineix interfícies de xarxa cablejades.
- `ens33` → nom de la interfície (pot variar: `eth0`, `enp0s3`, etc.).
- `dhcp4: true` → obté adreça IPv4 automàticament via DHCP.

Aquesta és la configuració típica per als clients de xarxa quan tenim un servidor DHCP.

Ubuntu pot utilitzar dos **renderers** principals:

- **systemd-networkd**: lleuger, orientat a servidors. Gestiona la xarxa via fitxers de configuració.
- **NetworkManager**: orientat a entorns d'escriptori. Més fàcil d'utilitzar amb interfície gràfica.

En entorns de servidor (com Ubuntu Server o WSL), normalment es fa servir `networkd`.

Vegem ara una configuració on assignem una IP manualment, típica del servidor:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.2.100/24
      gateway4: 192.168.2.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Intervenen dues interfícies (`enp0s3` i `enp0s8`): la primera es configura amb DHCP i la segona amb IP estàtica.

> **Important**: després de modificar un fitxer, cal aplicar els canvis amb `sudo netplan apply`. Si no surt cap missatge d'error, se suposa que tot ha anat bé, però sempre convé comprovar-ho.

Més informació sobre Netplan: [dani-cirvianum.gitbook.io/netplan](https://dani-cirvianum.gitbook.io/netplan).

### Sintaxi YAML bàsica

YAML significa *YAML Ain't Markup Language*. És un format de text senzill pensat per a fitxers de configuració. La idea principal és que funciona amb claus i valors:

```yaml
nom: Joan
edat: 20
```

La jerarquia es marca amb **espais**, mai amb tabuladors. Si s'usen malament els espais, el fitxer donarà error:

```yaml
persona:
  nom: Joan
  edat: 20
```

També permet llistes amb un guió davant de cada element:

```yaml
servidors_dns:
  - 8.8.8.8
  - 1.1.1.1
```

Els comentaris es posen amb el símbol `#`.

### Comprovació de la configuració de la xarxa

**1. Comprovar l'adreça IP.** Per veure les adreces IP de totes les interfícies:

```bash
ip a
```

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.1.50/24 brd 192.168.1.255 scope global ens33
```

Aquí tenim la IP `192.168.1.50` amb màscara `255.255.255.0`.

**2. Comprovar la porta d'enllaç (gateway).** La porta d'enllaç és el dispositiu que connecta la nostra xarxa local amb una altra xarxa, normalment Internet:

```bash
ip route
```

```
default via 192.168.1.1 dev ens33 proto dhcp
```

`default via 192.168.1.1` indica que la porta d'enllaç és el router `192.168.1.1`, i `dev ens33` la interfície que utilitza.

**3. Comprovar servidors DNS.** Els servidors DNS resolen noms de domini (`www.exemple.com`) a adreces IP. Ubuntu modern els gestiona amb `systemd-resolved`:

```bash
resolvectl status
```

```
nameserver 8.8.8.8
nameserver 1.1.1.1
```

En alguns sistemes es pot mirar també el fitxer simbòlic `/etc/resolv.conf` (que no s'ha d'editar directament).

## Servei DHCP

### Introducció

El servei **DHCP** (*Dynamic Host Configuration Protocol*) és un protocol que permet assignar automàticament configuracions de xarxa als ordinadors i dispositius que es connecten a una xarxa.

Quan un ordinador s'encén i es connecta a la xarxa, en comptes d'haver de configurar manualment l'adreça IP, la màscara, la porta d'enllaç i els servidors DNS, envia una petició al servidor DHCP. Aquest respon assignant-li una IP lliure i la resta de paràmetres necessaris.

El funcionament bàsic es coneix com el procés **DORA**:

- **Discover**: el client demana si hi ha algun servidor DHCP disponible.
- **Offer**: el servidor ofereix una adreça IP lliure.
- **Request**: el client accepta l'oferta i demana formalment aquesta configuració.
- **Acknowledge**: el servidor confirma i reserva aquesta IP per al client durant un temps determinat.

La gran avantatge del DHCP és que estalvia molta feina als administradors de xarxa i evita errors de configuració (IPs duplicades, oblits, etc.). Sense DHCP, cada dispositiu s'hauria de configurar manualment, cosa impracticable en xarxes amb molts usuaris.

Vídeos recomanats abans de continuar:

- [Vídeo 1 — funcionament del DHCP](https://drive.google.com/file/d/1976J5IVvYrUzVksZt78qBxYiiVkJVfwJ/view?usp=drive_link)
- [Vídeo 2 — DORA en detall](https://drive.google.com/file/d/1rP8EGWDTeidlosXi8LZJ4Q6yzTSnH6p5/view?usp=drive_link)

Abans d'entrar en la instal·lació i configuració del servei, cal veure les ordres i característiques comunes a tots els serveis de xarxa del curs.

## Gestió bàsica de serveis a Ubuntu

Quan treballem amb serveis de xarxa a Ubuntu (Apache2, DHCP, SSH, Samba...) hi ha un conjunt de conceptes i comandes bàsiques que es repeteixen. Entendre'ls és fonamental abans d'entrar en la configuració específica de cada servei.

### Instal·lació d'un servei

La majoria de serveis es troben als repositoris oficials d'Ubuntu i s'instal·len amb `apt`:

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

Ordres bàsiques:

- `apt update` → actualitza la llista de paquets disponibles.
- `apt install <paquet>` → instal·la el servei.
- `apt remove <paquet>` → elimina el servei (conservant configuració).
- `apt purge <paquet>` → elimina el servei i els fitxers de configuració.

### Gestió del servei amb systemctl

Ubuntu utilitza `systemd` per gestionar serveis. Les ordres són sempre les mateixes; només canvia el nom del servei. Exemples amb `apache2`:

```bash
sudo systemctl start apache2       # iniciar
sudo systemctl stop apache2        # aturar
sudo systemctl restart apache2     # reiniciar
sudo systemctl reload apache2      # recarregar configuració
systemctl status apache2           # veure estat
sudo systemctl enable apache2      # activar a l'inici
sudo systemctl disable apache2     # desactivar a l'inici
```

També podem fer servir l'ordre `service`:

```bash
sudo service apache2 restart
```

### Directori de configuració (/etc)

La configuració dels serveis de xarxa es troba habitualment dins de `/etc`:

- Apache → `/etc/apache2/`
- DHCP → `/etc/dhcp/`
- SSH → `/etc/ssh/`
- Samba → `/etc/samba/`

Després de modificar un fitxer dins `/etc`, sovint cal reiniciar o recarregar el servei.

### Fitxers de registre (logs)

Quan un servei no funciona, els missatges d'error solen estar als fitxers de registre dins `/var/log/`:

- Logs generals → `/var/log/syslog`
- Logs d'Apache → `/var/log/apache2/`

Per veure les últimes 40 línies del log general:

```bash
sudo tail -40 /var/log/syslog
```

> **És molt important consultar el syslog abans de preguntar al professor.**

Quan el syslog s'omple molt, es pot buidar:

```bash
sudo su
cat /dev/null > /var/log/syslog
```

També es poden veure missatges amb `journalctl`:

```bash
journalctl -u apache2
```

### Comandes útils generals

- `ps aux | grep <servei>` → comprovar si un servei s'està executant.
- `ss -tulpn` → veure quins ports estan oberts i quins serveis els fan servir.
- `ping <adreça>` → comprovar connectivitat.
- `systemctl list-units --type=service` → llistar serveis actius.

## Instal·lació i configuració del servidor DHCP (isc-dhcp-server)

Material de consulta: [help.ubuntu.com/community/isc-dhcp-server](https://help.ubuntu.com/community/isc-dhcp-server).

> **Avís**: `isc-dhcp-server` **ja no rep manteniment actiu** per part d'ISC des de 2022 — el seu substitut oficial és **Kea DHCP** (vegeu més avall). Tot i així, `isc-dhcp-server` continua sent el paquet més utilitzat en documentació i formació per la seva simplicitat, i és el que farem servir a la major part de les pràctiques d'aquest RA. Al Projecte 1 del mòdul utilitzarem Kea.

### Instal·lació

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

### Assignar IP estàtica a la interfície

Abans que el servei DHCP funcioni correctament, la interfície de xarxa des de la qual repartirà adreces ha de tenir una IP fixa. Si no, en intentar arrencar el servei donarà error.

Això es fa editant la configuració de xarxa (Netplan, `/etc/netplan/`) i assignant una IP estàtica a la interfície corresponent, tal com s'ha explicat abans.

### Configuració de /etc/default/isc-dhcp-server

En aquest fitxer s'especifica en quines interfícies escoltarà el servei:

```
INTERFACESv4="ens33"
```

Vídeo recomanat abans de continuar: [Configuració bàsica del servei](https://drive.google.com/file/d/1TZKaon7wdZzjqT7iWfv2RkDxhmTIOxgQ/view?usp=drive_link).

### Configuració principal: /etc/dhcp/dhcpd.conf

Aquí definim:

- Temps de lloguer (`default-lease-time`, `max-lease-time`).
- Subnet (adreça de la xarxa + màscara).
- Rangs d'IPs que el servidor pot assignar.
- Opcions per a la xarxa: router (gateway), DNS, broadcast, etc.

El fitxer que es crea quan s'instal·la el servei és pràcticament un manual amb exemples predefinits. Convé desar-lo abans de modificar-lo:

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bck
```

Ara podem cercar l'exemple que millor s'ajusti al que volem fer i modificar-ne només la part necessària.

Exemple típic:

```conf
default-lease-time 600;
max-lease-time 7200;

option subnet-mask 255.255.255.0;
option broadcast-address 192.168.1.255;
option routers 192.168.1.254;
option domain-name-servers 192.168.1.1, 192.168.1.2;
option domain-name "exemple.local";

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.10 192.168.1.100;
  range 192.168.1.150 192.168.1.200;
}
```

Les línies `option` fora del bloc `subnet` són **opcions globals** que s'apliquen a tots els àmbits. Si volem que una `option` sigui exclusiva d'un àmbit, s'ha de posar dins de les claus `{}`.

### Permisos

Possibles errors de permisos: per exemple, que el servei no pugui obrir `/etc/dhcp/dhcpd.conf` o el fitxer de leases. Cal comprovar que els fitxers tenen permisos i propietari correctes:

```bash
sudo chown root:root /etc/dhcp/dhcpd.conf
sudo chmod 644 /etc/dhcp/dhcpd.conf
```

### Gestió del servei

```bash
sudo systemctl start isc-dhcp-server
sudo systemctl stop isc-dhcp-server
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

També es pot usar `service`:

```bash
sudo service isc-dhcp-server start
```

Per veure les IPs que el servidor ha assignat, es consulta el fitxer de leases:

```bash
cat /var/lib/dhcp/dhcpd.leases
```

Aquí es guarda:

- Quina IP s'ha donat.
- A quin dispositiu (identificat per la MAC).
- Quan caduca el lloguer.
- L'estat (actiu, lliure, etc.).

### Exemple pràctic complet

Imaginem que tenim una xarxa local `192.168.50.0/24`. Volem que el servidor DHCP reparteixi adreces de la `.10` a la `.50`, que la interfície del servidor tingui IP fixa `192.168.50.1`, que el gateway sigui `192.168.50.254` i els DNS `8.8.8.8` i `8.8.4.4`.

**1. Assignar IP estàtica a la interfície (suposem `ens33`) via Netplan:**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      addresses:
        - 192.168.50.1/24
      gateway4: 192.168.50.254
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

```bash
sudo netplan apply
```

**2. Instal·lar isc-dhcp-server:**

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

**3. Editar `/etc/default/isc-dhcp-server`:**

```
INTERFACESv4="ens33"
```

**4. Editar `/etc/dhcp/dhcpd.conf`:**

```conf
default-lease-time 600;
max-lease-time 7200;

option subnet-mask 255.255.255.0;
option broadcast-address 192.168.50.255;
option routers 192.168.50.254;
option domain-name-servers 8.8.8.8, 8.8.4.4;
option domain-name "exemple.local";

subnet 192.168.50.0 netmask 255.255.255.0 {
  range 192.168.50.10 192.168.50.50;
}
```

**5. Comprovar permisos:**

```bash
sudo chown root:root /etc/dhcp/dhcpd.conf
sudo chmod 644 /etc/dhcp/dhcpd.conf
```

**6. Activar i arrencar el servei:**

```bash
sudo systemctl enable isc-dhcp-server
sudo systemctl start isc-dhcp-server
```

**7. Comprovar l'estat:**

```bash
sudo systemctl status isc-dhcp-server
```

**8. Provar des d'un client**: configura un dispositiu per obtenir IP automàticament i comprova que rep una IP dins del rang, i que es pot fer `ping` al gateway i a una IP externa.

## Reserves d'IP per MAC

Una **reserva** consisteix a assignar sempre la mateixa IP a un client concret, identificat per la seva adreça MAC. Serveix per a dispositius que han de tenir sempre la mateixa adreça (impressores, servidors interns, càmeres IP...) però sense renunciar a la gestió centralitzada del DHCP.

A `isc-dhcp-server` es fa amb un bloc `host` fora del `subnet` (les reserves s'apliquen globalment al servei, encara que la IP triada pertanyi a un àmbit concret):

```conf
host impressora-planta1 {
  hardware ethernet 00:11:22:33:44:55;
  fixed-address 192.168.50.20;
}
```

- `hardware ethernet` → MAC del client (es pot obtenir amb `ip a` a Linux o `ipconfig /all` a Windows).
- `fixed-address` → IP que se li reservarà. Ha d'estar dins del rang de la subxarxa configurada, però **es recomana que estigui fora del `range` dinàmic** per evitar conflictes.

Aquesta pràctica es desenvolupa a la Part 2 de `practiques.md`.

## Múltiples interfícies (multi-subnet)

Un mateix servidor DHCP pot servir diverses subxarxes alhora si té una interfície de xarxa per cada una. Cal:

1. Llistar totes les interfícies a `/etc/default/isc-dhcp-server`:

    ```
    INTERFACESv4="ens33 ens34"
    ```

2. Definir un bloc `subnet` per cada xarxa a `/etc/dhcp/dhcpd.conf`, cadascun amb el seu `range`, `option routers`, etc.

3. Assegurar-se que cada interfície té assignada una IP estàtica dins de la xarxa que servirà.

Aquesta configuració es practica en detall a la Part 2 de `practiques.md`.

## ISC-DHCP-Server vs. Kea

Kea és el substitut oficial d'`isc-dhcp-server` desenvolupat per ISC (la mateixa organització). A nivell funcional bàsic són equivalents (reparteixen IPs seguint el mateix procés DORA), però hi ha diferències importants:

| Aspecte | `isc-dhcp-server` | Kea |
|---|---|---|
| Estat del projecte | Sense manteniment actiu des de 2022 | Actiu, versió recomanada per ISC |
| Format de configuració | Sintaxi pròpia (`dhcpd.conf`) | JSON |
| Recàrrega en calent | No (cal reiniciar el servei) | Sí (via `kea-shell` o senyal) |
| Emmagatzematge de leases | Fitxer de text (`dhcpd.leases`) | Fitxer, MySQL, PostgreSQL o memòria |
| Alta disponibilitat | No integrada | Sí (mode *HA hook*) |
| API de gestió | No | Sí (REST/JSON opcional) |
| IPv6 | Servei separat (`isc-dhcp-server6`) | Integrat (`kea-dhcp6`) |

Exemple mínim de configuració Kea equivalent al `dhcpd.conf` de l'exemple pràctic anterior (`/etc/kea/kea-dhcp4.conf`):

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": ["ens33"]
    },
    "valid-lifetime": 600,
    "max-valid-lifetime": 7200,
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.50.0/24",
        "pools": [ { "pool": "192.168.50.10 - 192.168.50.50" } ],
        "option-data": [
          { "name": "routers", "data": "192.168.50.254" },
          { "name": "domain-name-servers", "data": "8.8.8.8, 8.8.4.4" }
        ]
      }
    ]
  }
}
```

A la Part 4 de `practiques.md` es fa una introducció mínima a Kea (instal·lació, migració d'un àmbit senzill i comprovació). El **Projecte 1** del mòdul utilitzarà Kea com a servidor DHCP definitiu.
