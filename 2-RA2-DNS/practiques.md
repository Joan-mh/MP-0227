# Pràctiques RA2 — DNS

> Al llarg de totes les pràctiques cal que un **client Windows** i un **client Linux** obtinguin els resultats esperats. Configureu-los perquè utilitzin el vostre servidor DNS com a primari i valideu la resolució amb `nslookup` (Windows) o `dig`/`nslookup` (Linux).
>
> El **domini d'exemple** és `cognom.edu`: cada alumne el substitueix pel seu propi cognom (per exemple, `mercade.edu`). Si preferiu, podeu triar qualsevol TLD reservat per a proves (`.test`, `.example`, `.local`) — veure la teoria per a la llista.
>
> **La xarxa** és `192.168.X.0/24`: el tercer octet `X` el trieu vosaltres (mantingueu-lo coherent en totes les parts).

## Part 1 — Instal·lació i configuració bàsica de bind9 amb zona directa i forwarders (2 hores)

### Objectius

- Instal·lar i comprovar el funcionament de `bind9`.
- Configurar una **zona directa** amb un domini inventat (`cognom.edu`).
- Validar la configuració amb **`named-checkzone`**.
- Configurar **forwarders** per a resolució de noms externs.
- Fer que un client **Linux i un client Windows** utilitzin el servidor com a DNS primari.

### Tasques

#### 1. Instal·lació

1. Instal·la `bind9`, `bind9utils` i `bind9-doc`.
2. Comprova que el servei arrenca correctament (`systemctl status bind9`).

#### 2. Declaració de la zona i forwarders

1. Edita `/etc/bind/named.conf.local` i afegeix la zona `cognom.edu` (tipus `master`, fitxer `/etc/bind/db.cognom.edu`).
2. Edita `/etc/bind/named.conf.options` i configura almenys dos **forwarders** (per exemple, `1.1.1.1` i `8.8.8.8`).

#### 3. Fitxer de zona directa

Crea `/etc/bind/db.cognom.edu` amb, com a mínim:

- Un registre **SOA** correcte (respecta el format i el punt final dels FQDN).
- Un registre **NS** apuntant al servidor de noms.
- Un registre **A** per a `ns1` (IP del servidor).
- Un registre **A** per a `www` (IP inventada dins de la mateixa xarxa).

#### 4. Validació al servidor

1. Executa `sudo named-checkconf`.
2. Executa `sudo named-checkzone cognom.edu /etc/bind/db.cognom.edu`.
3. Reinicia bind9 i comprova l'estat.

#### 5. Configuració dels clients

1. Configura el **client Linux** perquè el seu DNS primari sigui la IP del servidor (via Netplan o el gestor de xarxa).
2. Configura el **client Windows** de la mateixa manera (Configuració de xarxa → adaptador → Propietats de TCP/IPv4).

#### 6. Validació des dels clients

Des del **Linux**:

```bash
dig @IP_SERVIDOR ns1.cognom.edu
dig www.cognom.edu
dig www.google.com          # ha de resoldre gràcies als forwarders
```

Des del **Windows**:

```
nslookup ns1.cognom.edu
nslookup www.cognom.edu
nslookup www.google.com
```

### Resultats esperats

- El client Linux i el Windows resolen `ns1.cognom.edu` i `www.cognom.edu` a les IPs configurades.
- Tots dos clients resolen també noms externs (per exemple, `www.google.com`) gràcies als forwarders.
- `named-checkzone` retorna `OK`.

---

## Part 2 — Zones inverses i registres addicionals (2 hores)

### Objectius

- Crear i configurar una **zona inversa** IPv4.
- Afegir registres addicionals: **CNAME** i **MX**.
- Validar amb proves de resolució directa i inversa des de Linux i Windows.
- **Extra opcional**: introduir una zona inversa **IPv6** (`ip6.arpa`).

### Tasques

#### 1. Zona inversa IPv4

1. Afegeix una zona inversa a `/etc/bind/named.conf.local`. Per a la xarxa `192.168.X.0/24`, el nom serà `X.168.192.in-addr.arpa`.
2. Crea el fitxer `/etc/bind/db.192.168.X` amb:
   - **SOA** i **NS** anàlegs a la zona directa.
   - Un **PTR** per a `ns1`.
   - Un **PTR** per a `www`.

#### 2. Registres addicionals a la zona directa

Afegeix a `db.cognom.edu`:

- Un **CNAME**: `ftp` → `www` (simula que el servidor de FTP i el web són el mateix host).
- Un **MX**: apuntant a `mail.cognom.edu.` amb preferència `10`, i el corresponent registre `A` per a `mail`.

> **No oblidis incrementar el serial** del SOA cada cop que modifiquis la zona.

#### 3. Validació i comprovacions

Al servidor:

```bash
sudo named-checkzone cognom.edu /etc/bind/db.cognom.edu
sudo named-checkzone X.168.192.in-addr.arpa /etc/bind/db.192.168.X
sudo systemctl restart bind9
```

Des del **client Linux**:

```bash
dig ftp.cognom.edu                   # ha de mostrar el CNAME i la A de www
dig cognom.edu MX                    # ha de mostrar el registre MX
dig -x 192.168.X.10                  # ha de retornar www.cognom.edu.
```

Des del **client Windows**:

```
nslookup ftp.cognom.edu
nslookup -type=MX cognom.edu
nslookup 192.168.X.10
```

#### 4. Extra opcional — zona inversa IPv6

Aquesta part és **opcional** però recomanada si vas amb temps.

1. Assigna una adreça IPv6 (per exemple, una ULA `fd00:0:0:X::1/64`) al servidor via Netplan.
2. Afegeix un registre **AAAA** per a `ns1` a la zona directa.
3. Declara una zona inversa `ip6.arpa` a `named.conf.local`. Recorda: la IP s'escriu **nibble a nibble** i **al revés**. Per exemple, `fd00:0:0:X::1` inversa: `1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.X.0.0.0.0.0.0.0.0.0.0.0.0.0.d.f.ip6.arpa`.
4. Crea el fitxer de zona corresponent amb un `PTR` per a `ns1`.
5. Comprova amb `dig -x fd00:0:0:X::1`.

> Consell: hi ha eines com `ipv6calc --in ipv6 fd00::1 --out revnibbles.arpa` o pàgines web que generen el nom de la zona inversa automàticament.

### Resultats esperats

- Resolució inversa `IP → nom` funciona des de Linux i Windows per a `ns1` i `www`.
- `ftp.cognom.edu` resol a la IP de `www` via CNAME.
- `dig cognom.edu MX` mostra el registre MX i la A associada al mail.
- (Opcional) La resolució inversa IPv6 funciona per a `ns1`.

---

## Part 3 — Servidor DNS secundari (2 hores)

### Objectius

- Configurar un **servidor secundari (slave)** per a la zona `cognom.edu`.
- Entendre el funcionament de la **transferència de zona**.
- Comprovar la redundància fent que els clients Linux i Windows apuntin **primer al master i després al slave**.

### Preparació

Cal disposar de **dues VMs a la mateixa xarxa**:

- **`ns1`** (master): el servidor ja configurat a les parts 1 i 2.
- **`ns2`** (slave): una nova VM Ubuntu amb bind9 acabat d'instal·lar.

Anota les IPs. Als exemples faig servir `192.168.X.1` (master) i `192.168.X.2` (slave).

### Tasques

#### 1. Autoritzar la transferència al master

Al `named.conf.local` del **master**, dins del bloc de la zona `cognom.edu`, afegeix `allow-transfer` amb la IP del slave. Fes el mateix a la zona inversa.

Afegeix també un registre **NS** per a `ns2` i el seu **A** al fitxer `db.cognom.edu` (recorda incrementar el serial).

Reinicia bind9 al master.

#### 2. Declarar la zona al slave

Al `named.conf.local` del **slave**, declara les zones `cognom.edu` i `X.168.192.in-addr.arpa` com a `type slave` (o `secondary`) i indica la IP del master amb `masters { ... };`.

Reinicia bind9 al slave.

#### 3. Comprovar la transferència de zona

Al **slave**, comprova que els fitxers de zona s'han copiat automàticament:

```bash
sudo ls -l /var/cache/bind/
sudo journalctl -u bind9 -n 30
```

Hauries de veure els fitxers `db.cognom.edu` (o similar) descarregats del master i missatges de log tipus `zone cognom.edu/IN: transferred serial X`.

#### 4. Validació creuada des dels clients

Des del **Linux** i des del **Windows**, fes consultes tant al master com al slave:

```bash
dig @192.168.X.1 www.cognom.edu       # master
dig @192.168.X.2 www.cognom.edu       # slave
```

```
nslookup www.cognom.edu 192.168.X.1
nslookup www.cognom.edu 192.168.X.2
```

Les respostes han de ser **idèntiques**.

#### 5. Sincronització de canvis

1. Al master, afegeix un nou registre a `db.cognom.edu` (per exemple, `intranet IN A 192.168.X.30`).
2. **Incrementa el serial** del SOA.
3. Reinicia bind9 al master (o executa `sudo rndc reload cognom.edu`).
4. Al slave, comprova al cap d'uns segons que la nova entrada està disponible:

```bash
dig @192.168.X.2 intranet.cognom.edu
```

Si el slave no ha refrescat, pots forçar-ho amb `sudo rndc retransfer cognom.edu`.

#### 6. Redundància

1. Configura els dos clients (Linux i Windows) perquè tinguin `192.168.X.1` com a DNS primari i `192.168.X.2` com a secundari.
2. Atura el servei al master (`sudo systemctl stop bind9`).
3. Comprova que els clients continuen podent resoldre noms.

### Resultats esperats

- El slave rep la zona automàticament del master (transferència AXFR/IXFR visible al log).
- Master i slave donen la mateixa resposta per a les mateixes consultes.
- Els canvis fets al master es propaguen al slave en incrementar el serial.
- Aturant el master, els clients continuen resolent gràcies al secundari.
