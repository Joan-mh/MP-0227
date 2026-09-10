# Pt0 — Preparació de l'entorn

Aquesta pràctica **no puntua** i no pertany a cap RA, però és **obligatòria** per poder fer la resta del curs. L'objectiu és tenir a punt totes les eines i les màquines virtuals que utilitzarem durant l'any.

## Objectius

- Instal·lar les eines de treball al vostre equip amfitrió (host).
- Crear i configurar les màquines virtuals del curs a VirtualBox.
- Entendre els diferents tipus d'adaptadors de xarxa de VirtualBox.
- Comprovar que les màquines es "veuen" entre elles i que podeu controlar el servidor per SSH des del host.

## Eines a instal·lar al host

### VirtualBox (obligatori)

Descarregueu i instal·leu [VirtualBox](https://www.virtualbox.org/wiki/Downloads) per al vostre sistema operatiu. Instal·leu també el **Extension Pack** (mateixa pàgina) — permet USB 2.0/3.0 i RDP integrat.

### Git (obligatori)

- **Windows**: instal·leu [Git for Windows](https://git-scm.com/download/win) (inclou Git Bash).
- **Linux**: `sudo apt install git`.
- **macOS**: `brew install git` o instal·leu les Xcode Command Line Tools.

Configureu el vostre nom i correu:

```bash
git config --global user.name "El vostre nom"
git config --global user.email "el.vostre@correu.cat"
```

### Compte GitHub (recomanable)

No és obligatori, però és **molt recomanable** obrir un compte a [github.com](https://github.com) per publicar-hi les vostres pràctiques i tenir un porfoli. És gratuït.

### Obsidian (recomanable)

No és obligatori, però [Obsidian](https://obsidian.md) és una eina excel·lent per prendre apunts en Markdown (el mateix format en què està escrit tot el material del curs). Us permetrà tenir els vostres apunts organitzats i enllaçats.

## Màquines virtuals del curs

Farem servir **tres** màquines virtuals a VirtualBox:

| Rol | Sistema operatiu | Notes |
|---|---|---|
| Servidor | **Ubuntu Server 24.04 LTS** o superior | La màquina on configurarem tots els serveis. |
| Client Linux | Ubuntu Desktop (o qualsevol distribució Linux) | Les guies estan pensades per Ubuntu, però podeu triar. |
| Client Windows | **Windows 10 o 11** | Windows 7 **no recomanat** (problemes amb certificats i navegadors moderns). |

> **Nota**: al curs anterior es feia servir també un Windows Server. Enguany **no** el fem servir, perquè cap pràctica el requereix.

### Recomanacions generals al crear cada VM

- **Disc dinàmic**: no reserva tot l'espai de cop, només el que necessita.
- Si treballeu amb un portàtil sense gaire disc, guardeu els discs de les VMs en un **disc dur extern**.
- RAM recomanada: 2 GB per Ubuntu Server, 4 GB per Ubuntu Desktop i Windows.

## Adaptadors de xarxa a VirtualBox

Aquest és el punt més important d'aquesta preparació. Configurar malament la xarxa **pot interferir amb la xarxa del centre** quan al llarg del curs muntem serveis com DHCP o DNS.

VirtualBox ofereix diversos modes d'adaptador. Els que farem servir són:

### NAT

La VM surt a Internet **a través del host**, com si fos una aplicació més. VirtualBox li dóna una IP privada (normalment 10.0.2.15) i fa NAT.

- Cap altra VM ni el host la pot "veure" directament.
- Serveix bàsicament per **actualitzar paquets** i descarregar programari.

### Xarxa interna (*Internal Network*)

Una xarxa **totalment aïllada** entre VMs. Les VMs que estan a la mateixa xarxa interna es veuen entre elles, però ni el host ni Internet hi tenen accés.

- Ideal per simular la LAN interna del nostre "laboratori", **sense risc** d'interferir amb la xarxa del centre.
- Cal donar-li un **nom** (per exemple `intnet-lab`) i totes les VMs que hi hagin de connectar han de fer servir el mateix nom.

### Host-only (*Adaptador només amfitrió*)

Una xarxa privada entre el **host i les VMs**. El host veu les VMs (per SSH, RDP, navegador...) i les VMs veuen el host, però no surten a Internet.

- Perfecta per **controlar el servidor amb SSH** des del vostre sistema operatiu principal.
- A `Fitxer → Eines → Gestor de xarxa` de VirtualBox podeu veure/crear la xarxa host-only (normalment `vboxnet0`).

### Adaptador pont (*Bridge*)

La VM apareix a la xarxa física del host **com si fos un ordinador més**. Rep IP del router del centre.

- **No la farem servir en general** perquè xocaria amb la xarxa del centre.
- Sí que la farem servir puntualment al RA7 (Wifi), amb un punt d'accés propi.

## Configuració concreta d'adaptadors per VM

### Ubuntu Server (3 adaptadors)

| Adaptador | Mode | Nom | Ús |
|---|---|---|---|
| 1 | NAT | — | Sortida a Internet (actualitzacions, `apt`). |
| 2 | Xarxa interna | `intnet-lab` | Servir els clients (DHCP, DNS, HTTP...). |
| 3 | Només amfitrió | `vboxnet0` | Controlar el servidor per SSH des del host. |

### Clients (Ubuntu Desktop i Windows)

| Adaptador | Mode | Nom | Ús |
|---|---|---|---|
| 1 | Xarxa interna | `intnet-lab` | Connectar-se al servidor. |

Els clients **no necessiten sortida a Internet** durant les pràctiques. Si en algun moment els cal (per exemple, per instal·lar un navegador), podeu canviar temporalment l'adaptador a NAT i tornar-lo a xarxa interna quan acabeu.

## Instal·lació dels sistemes operatius

Descarregueu les ISOs oficials, carregueu-les a la unitat òptica de la VM i inicieu-la. Seguiu l'assistent d'instal·lació.

### Ubuntu Server: components a instal·lar

Durant la instal·lació us demanarà quins components (snaps) voleu instal·lar. **Deixeu-los tots en blanc** — els instal·larem manualment amb `apt` a mesura que els necessitem al llarg del curs. Això manté el sistema net.

L'única excepció recomanada és marcar **OpenSSH Server**, així ja podreu connectar-vos al servidor per SSH des del host des del primer moment. Si no el marqueu, també el podreu instal·lar després amb:

```bash
sudo apt install openssh-server
```

### Windows: desactivar el tallafoc

Al nostre laboratori podem **desactivar el tallafoc de Windows** sense problema. És necessari perquè per defecte **bloqueja els `ping` (ICMP)** i no podríeu comprovar la connectivitat.

## Guest Additions

Un cop instal·lats els SOs, instal·leu les **Guest Additions** de VirtualBox — milloren rendiment, resolució de pantalla i integració amb el host.

- **Ubuntu Desktop / Windows**: menú `Dispositius → Inserir imatge del CD de Guest Additions` i seguir l'assistent.
- **Ubuntu Server** (sense entorn gràfic):

```bash
sudo apt update
sudo apt install build-essential dkms linux-headers-$(uname -r)
sudo mount /dev/cdrom /mnt
sudo /mnt/VBoxLinuxAdditions.run
```

## Verificació final

Un cop tot instal·lat, comproveu:

1. **Adreces IP manuals** a tots els adaptadors de xarxa interna (per exemple, el servidor a `192.168.100.1/24` i els clients a `192.168.100.10/24`, `192.168.100.11/24`...). Trieu vosaltres el rang.
2. **Ping** entre les tres VMs a través de la xarxa interna.
3. **SSH del host al servidor** per la xarxa host-only:

```bash
ssh usuari@<ip-host-only-del-servidor>
```

4. El servidor té **sortida a Internet** (`ping 8.8.8.8` i `apt update` funcionen).

Si tot això respon correctament, l'entorn està a punt per començar el curs.
