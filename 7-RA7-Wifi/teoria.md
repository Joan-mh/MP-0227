# Recordatori de TCP/IP i introducció al wifi

Aquest RA obre el mòdul amb un **recordatori** ràpid dels fonaments de TCP/IP que ja heu vist a mòduls anteriors, però que són imprescindibles per a la resta del curs (DHCP, DNS, FTP, etc.). Aprofitem el punt d'accés wifi físic del laboratori per repassar la teoria amb un exemple viu.

> Aquest RA no té examen escrit; s'avalua **en directe durant la pràctica** a partir del `checklist.md` associat.

## Conceptes fonamentals

En el món de les xarxes utilitzem terminologia sovint derivada de l'anglès. Els components bàsics:

- **Adreça IP (IP address)**: identificador únic de 32 bits (en IPv4) que té cada dispositiu en una xarxa. Permet que les dades sàpiguen cap a on han d'anar.
- **Màscara de xarxa (subnet mask)**: determina quina part de l'adreça IP correspon a la xarxa i quina part al *host* (el dispositiu). Sense la màscara, l'ordinador no sap si una altra IP està a la seva pròpia xarxa o a una externa.
- **Porta d'enllaç (default gateway)**: adreça del dispositiu (normalment un encaminador o *router*) que connecta la nostra xarxa local amb altres xarxes o amb Internet. Si un paquet va a una IP que no és de la nostra xarxa, s'envia aquí.
- **Servidors DNS**: encarregats de traduir noms de domini (`www.google.com`) en adreces IP. Sense ells, hauríem de recordar les IPs de tots els llocs web.

## L'estructura de l'adreça IP

Dins d'un rang de xarxa hi ha dues adreces que **mai** es poden assignar a un dispositiu:

- **Adreça de xarxa**: la primera adreça del rang. Identifica la xarxa en si. Exemple: en `192.168.1.0/24`, l'adreça de xarxa és `192.168.1.0`.
- **Adreça de difusió (broadcast)**: l'última adreça del rang. S'utilitza per enviar un missatge a **tots** els dispositius de la mateixa xarxa simultàniament. Exemple: en `192.168.1.0/24`, és la `192.168.1.255`.

## Classes d'adreces IP (històric)

Actualment s'utilitza CIDR (màscares variables), però és útil recordar les classes clàssiques:

| Classe | Rang del primer octet | Màscara per defecte | Ús típic |
|---|---|---|---|
| **A** | 1 – 126 | 255.0.0.0 (/8) | Xarxes molt grans (governs, multinacionals) |
| **B** | 128 – 191 | 255.255.0.0 (/16) | Xarxes mitjanes (universitats, empreses) |
| **C** | 192 – 223 | 255.255.255.0 (/24) | Xarxes petites (casa, oficines) |
| **D** | 224 – 239 | — | Multicast (grups de dispositius) |
| **E** | 240 – 255 | — | Investigació i experimentació |

L'adreça `127.x.x.x` està reservada per al **localhost** (loopback).

## IP pública vs. IP privada

Hi ha rangs reservats per a ús intern (LAN), no accessibles directament des d'Internet:

- `10.0.0.0` – `10.255.255.255`
- `172.16.0.0` – `172.31.255.255`
- `192.168.0.0` – `192.168.255.255`

Les **IPs públiques** són adreces úniques al món; ens les assigna el proveïdor de serveis (ISP).

## Comandes de comprovació

Al llarg del curs farem servir aquestes ordres per validar la configuració de xarxa dels clients:

Al Linux:

```bash
ip a              # veure IPs de les interfícies
ip route          # veure la porta d'enllaç
resolvectl status # veure servidors DNS
ping 192.168.1.1  # provar connectivitat al gateway
```

Al Windows (`cmd`):

```
ipconfig /all
ping 192.168.1.1
nslookup www.google.com
```

## Wifi i cable: la mateixa xarxa a nivell lògic

Aquest és el punt pedagògic central de l'RA7: **wifi i cable són equivalents a nivell lògic**. Un cop un dispositiu té una IP, una màscara i una porta d'enllaç, li és indiferent si ha rebut les dades per un cable de coure o per l'aire. Els protocols de nivell superior (IP, TCP, UDP, HTTP, DNS, DHCP...) són **exactament els mateixos**.

La diferència és **exclusivament física** i afecta com viatja la senyal:

- **Cable Ethernet**: parells trenats de coure, senyal elèctrica, connectors RJ-45, medi guiat.
- **Wifi (IEEE 802.11)**: ones de ràdio a 2,4 GHz o 5 GHz, medi compartit i no guiat, amb tot el que això comporta (interferències, cobertura, seguretat basada en clau WPA2/WPA3...).

Un servidor DHCP, per exemple, no distingeix entre clients cablejats i clients wifi. Els assigna IPs seguint el mateix protocol DORA (que veurem a l'RA1).

## Els adaptadors de xarxa a VirtualBox

Quan treballem amb màquines virtuals, VirtualBox ens ofereix diversos modes d'adaptador de xarxa. Els que farem servir al mòdul:

| Mode | Què fa | Quan el fem servir |
|---|---|---|
| **NAT** | La VM veu Internet a través del host, però és invisible per a la resta de la LAN | Quan només volem accés a Internet des de la VM |
| **Adaptador pont** (*Bridged*) | La VM apareix a la LAN com un dispositiu més, amb IP pròpia del rang de la xarxa física | Quan volem que la VM parli amb altres equips reals (com un AP wifi o un servidor DHCP físic) |
| **Xarxa interna** | Diverses VMs connectades entre elles, però aïllades del host i d'Internet | Laboratoris tancats amb topologies pròpies |
| **Host-only** | La VM parla amb el host i amb altres VMs host-only, però no amb la LAN externa | Practicar SSH i serveis interns entre servidor i client sense contaminar la xarxa (RA6) |

### Adaptador pont: escollir quin adaptador físic

Quan escollim mode **pont** a VirtualBox, ens demana **a través de quin adaptador físic** del host ha de fer el pont:

- Si triem l'**adaptador Ethernet** (per exemple `eno1` o similar), la VM apareixerà com un dispositiu més a la xarxa cablejada a la qual el host està connectat.
- Si triem l'**adaptador wifi** (per exemple `wlp3s0` o similar), la VM apareixerà com un dispositiu més a la xarxa wifi a la qual el host està connectat.

En els dos casos la VM rebrà una IP del mateix DHCP —el del router o el del punt d'accés wifi— i podrà parlar amb qualsevol altre equip de la mateixa xarxa.

**Aquest és el fet que confirmarem experimentalment a l'RA7**: connectarem VMs de Linux i Windows a través del pont, primer per Ethernet i després per wifi, i veurem que reben IP i es comuniquen exactament igual.

## El punt d'accés wifi

Un **punt d'accés (AP)** és el dispositiu que fa d'enllaç entre la xarxa wifi i una xarxa cablejada. Molts APs comercials, com el D-Link del laboratori, integren funcions addicionals:

- **Encaminament (routing)**: si es connecta a una xarxa externa amb sortida a Internet, fa de porta d'enllaç per als seus clients wifi.
- **Servidor DHCP**: assigna automàticament IP, màscara, gateway i DNS als clients que s'hi connecten.
- **DNS proxy**: reenvia les consultes DNS dels clients cap a un DNS extern.
- **Tallafoc bàsic**: separa la xarxa wifi de la xarxa externa amb regles NAT.

En aquest RA farem servir l'AP D-Link com a **servidor DHCP integrat** i veurem que, des del punt de vista dels clients, no hi ha cap diferència amb qualsevol altre servidor DHCP: reben IP i naveguen normalment.

## Recursos

- [Address Resolution Protocol (Wikipedia)](https://ca.wikipedia.org/wiki/Address_Resolution_Protocol)
- [Wikipedia — IPv4](https://ca.wikipedia.org/wiki/IPv4)
- Manual VirtualBox: capítol *Virtual networking* — [www.virtualbox.org/manual/](https://www.virtualbox.org/manual/)
