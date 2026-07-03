# Pràctiques RA1 — DHCP

## Part 1 — Instal·lació i configuració bàsica d'un servidor DHCP

### Objectius

- Instal·lar i configurar un servidor DHCP a Ubuntu.
- Definir un àmbit de xarxa de classe C (192.168.x.0/24, on l'alumne inventarà el tercer octet).
- Validar que un client de la xarxa rep una adreça IP automàticament.
- Comprovar quin servidor DHCP ha assignat l'adreça al client.

### Tasques a realitzar

#### 1. Instal·lació del servei

1. Actualitza els repositoris i instal·la el paquet `isc-dhcp-server`.

#### 2. Configuració del servidor DHCP

1. Edita el fitxer `/etc/default/isc-dhcp-server` i indica la interfície de xarxa on escoltarà.
2. Edita el fitxer `/etc/dhcp/dhcpd.conf` i crea un àmbit de classe C inventat.
3. Comprova que no hi hagi errors al syslog.
4. Reinicia el servei i comprova que està actiu.

#### 3. Configuració i prova al client

1. Configura el client perquè obtingui IP automàticament (DHCP).
   - A Linux
   - A Windows
2. Comprova al client quina IP ha rebut (`ip a` o `ipconfig`).
3. Verifica la connectivitat amb un `ping` al router/gateway.

#### 4. Identificar el servidor DHCP

1. Des del client, comprova quin servidor DHCP li ha donat l'adreça:
   - A Linux
   - A Windows
2. Confirma que és la IP del servidor que heu configurat.

### Resultats esperats

- El servidor DHCP assigna IPs dins del rang configurat.
- El client obté una adreça IP automàticament.

---

## Part 2 — Configuració d'un servidor DHCP amb dos àmbits (classe C i classe B)

### Objectius

- Configurar un servidor DHCP amb dues interfícies de xarxa.
- Definir dos àmbits: un de classe C i un de classe B.
- Verificar que clients de xarxes diferents reben correctament la configuració.
- Revisar el fitxer de leases i comprovar quins clients han rebut adreces.

### Tasques a realitzar

#### 1. Preparació del servidor

1. Assigna dues interfícies de xarxa al servidor (per exemple, `ens33` i `ens34`).
   - Una per a l'àmbit de classe C.
   - L'altra per a l'àmbit de classe B.
2. Configura cada interfície amb una adreça IP estàtica corresponent.

#### 2. Configuració del servei DHCP

1. Edita `/etc/default/isc-dhcp-server` i indica les dues interfícies.
2. Edita `/etc/dhcp/dhcpd.conf` i defineix dos àmbits.
3. Reinicia el servei i comprova que arrenca correctament.

#### 3. Validació amb clients

1. Connecta un client Linux a la xarxa de classe C i comprova que rep una IP correcta.
2. Connecta un client Windows a la xarxa de classe B i comprova que rep una IP correcta.
3. En tots dos casos:
   - Fer `ping` al gateway corresponent.
   - Fer `ping` a un servidor extern (ex: 8.8.8.8) si hi ha sortida a Internet.

#### 4. Reserves d'IP

1. Edita el fitxer `/etc/dhcp/dhcpd.conf` i afegeix una reserva a cada àmbit, que correspongui amb la MAC dels clients anteriors.
2. Reinicia el servei DHCP.
3. Configura els clients simulats perquè obtinguin IP automàticament i comprova que reben la IP reservada.

#### 5. Revisió de leases

1. Obre i revisa el fitxer `/var/lib/dhcp/dhcpd.leases`.
2. Identifica:
   - Les entrades actives per a cada client.
   - Les entrades de les reserves.
   - El temps de lease i l'estat de cada entrada.

### Resultats esperats

- El servidor DHCP reparteix adreces correctament en dos àmbits diferents.
- Els clients Linux i Windows obtenen IPs segons la seva xarxa.
- Els clients obtenen la IP reservada.

---

## Part 3 — Opcions DHCP avançades: globals vs. per àmbit

### Objectius

- Diferenciar clarament **opcions globals** (fora de qualsevol `subnet`) de **opcions específiques d'àmbit** (dins de `{}`).
- Configurar opcions DHCP addicionals: nom de domini, servidors NTP, i sobreescriure DNS per un àmbit concret.
- Comprovar des del client que ha rebut cada paràmetre correctament.

### Tasques a realitzar

#### 1. Punt de partida

Partim de la configuració de la Part 2 (dos àmbits: classe C i classe B), amb el servei arrencant correctament.

#### 2. Afegir opcions globals

A `/etc/dhcp/dhcpd.conf`, **fora** de qualsevol bloc `subnet`, afegeix les opcions següents perquè s'apliquin a tots dos àmbits:

- `option domain-name` amb un domini inventat (per exemple, `curs.local`).
- `option ntp-servers` amb almenys un servidor NTP públic (`pool.ntp.org` no serveix — cal una IP; podeu usar, per exemple, `162.159.200.1` de Cloudflare).
- `option domain-name-servers` amb `1.1.1.1` i `8.8.8.8`.

#### 3. Sobreescriure una opció per àmbit

Dins del bloc `subnet` de la xarxa de classe B, **redefineix** `option domain-name-servers` amb un valor diferent (per exemple, `9.9.9.9`). Comprova que la global s'aplica a la classe C i la local a la classe B.

#### 4. Reiniciar i validar

1. Reinicia el servei i comprova que arrenca sense errors.
2. Renova la concessió des d'un client de cada àmbit.
3. Comprova al client que s'han rebut els paràmetres esperats:
   - A Linux: `resolvectl status` i `cat /var/lib/dhcp/dhclient.leases`.
   - A Windows: `ipconfig /all`.

### Resultats esperats

- Les opcions globals arriben a tots els clients.
- L'opció redefinida dins d'un `subnet` pren prioritat sobre la global per als clients d'aquell àmbit.
- L'alumne sap explicar per què i quan s'utilitza cada nivell d'opció.

---

## Part 4 — Introducció a Kea DHCP

> **Nota**: aquesta pràctica no substitueix el Projecte 1 (on Kea es fa servir en profunditat). Aquí només s'instal·la, es fa arrencar amb una configuració mínima i es comprova que reparteix IPs — perquè al projecte no s'hi arribi de zero.

### Objectius

- Desinstal·lar (o desactivar) `isc-dhcp-server` per evitar conflictes de port.
- Instal·lar `kea-dhcp4-server`.
- Escriure una configuració mínima en format JSON equivalent a la de la Part 1.
- Comprovar que Kea assigna IP a un client de la mateixa manera que ho feia `isc-dhcp-server`.

### Tasques a realitzar

#### 1. Desactivar isc-dhcp-server

Kea i `isc-dhcp-server` utilitzen el mateix port UDP 67. Cal aturar i desactivar el servei anterior abans d'arrencar Kea:

```bash
sudo systemctl stop isc-dhcp-server
sudo systemctl disable isc-dhcp-server
```

Opcionalment, si vols eliminar-lo completament del sistema (recordant que conservar la configuració pot ser útil per comparar):

```bash
sudo apt remove isc-dhcp-server    # elimina el paquet, conserva /etc/dhcp/
# o bé
sudo apt purge isc-dhcp-server     # elimina també la configuració
```

#### 2. Instal·lar Kea

```bash
sudo apt update
sudo apt install kea-dhcp4-server
```

El fitxer principal de configuració és `/etc/kea/kea-dhcp4.conf` (format JSON). Desa'n una còpia abans de modificar-lo.

#### 3. Configuració mínima

Substitueix el contingut de `/etc/kea/kea-dhcp4.conf` per una configuració equivalent a la Part 1 (classe C, un àmbit senzill). La sintaxi és JSON — consulta l'exemple de la secció «ISC-DHCP-Server vs. Kea» de `teoria.md`.

Elements que ha de tenir la configuració:

- Interfície on escoltarà (`interfaces-config`).
- Temps de lease (`valid-lifetime`, `max-valid-lifetime`).
- Un `subnet4` amb `subnet`, `pools` i `option-data` (routers + DNS).

#### 4. Arrencar i comprovar

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Des d'un client de l'àmbit configurat, renova la IP i comprova que rep una adreça correcta i pot fer `ping` al gateway.

#### 5. Comparació ràpida

Anota 2 o 3 diferències que hagis notat entre `isc-dhcp-server` i Kea (format del fitxer, missatges de log, ubicació del fitxer de leases a `/var/lib/kea/`, etc.).

### Resultats esperats

- `isc-dhcp-server` desactivat, Kea actiu i escoltant al port 67.
- Un client rep IP via DHCP servit per Kea.
- L'alumne sap identificar les diferències bàsiques de configuració i preparar-se per al Projecte 1.
