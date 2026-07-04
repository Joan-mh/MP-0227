# Pràctiques RA7 — Wifi i adaptador pont

> **Material del laboratori**:
>
> - Punt d'accés wifi **D-Link** (amb servidor DHCP integrat).
> - Un switch físic si convé, per compartir l'AP entre diversos alumnes.
> - PC amb VirtualBox instal·lat i **dues màquines virtuals**: una amb Ubuntu Desktop i una altra amb Windows 10/11.
> - Connexió bàsica a Internet (opcional, per validar DNS i gateway).
>
> **Idea general**: comprovar experimentalment que wifi i cable són equivalents a nivell lògic. Ho farem en tres passes de dificultat creixent.

## Part 1 — Adaptador pont a VirtualBox (cablejat)

### Objectius

- Configurar dues VMs (Ubuntu Desktop i Windows) amb **mode pont** a VirtualBox, connectades a l'adaptador **Ethernet** del host.
- Comprovar que les dues VMs reben una IP del router/AP de l'aula, com si fossin dispositius físics de la LAN.
- Verificar la connectivitat entre les dues VMs i cap a Internet.

### Tasques

#### 1. Configurar el mode pont a les dues VMs (cablejat)

1. Al VirtualBox, atura les dues VMs (Ubuntu i Windows).
2. A cadascuna, ves a **Configuració → Xarxa → Adaptador 1**:
   - **Enable Network Adapter**: activat.
   - **Attached to**: `Bridged Adapter`.
   - **Name**: escull l'**adaptador Ethernet físic** del host (per exemple `eno1`, `enp3s0`, ... — el nom depèn del PC).
3. Deixa la resta de valors per defecte.

#### 2. Arrencar i comprovar la IP rebuda

1. Arrenca les dues VMs.
2. Al **Ubuntu**, obre una terminal i executa:

   ```bash
   ip a
   ip route
   ```

3. Al **Windows**, obre `cmd` i executa:

   ```
   ipconfig /all
   ```

4. Anota per a cada VM: IP rebuda, màscara, porta d'enllaç i servidor DNS.
5. Compara amb la IP del **host** (des d'un altre terminal del host: `ip a` a Linux o `ipconfig` a Windows).

    Totes tres han d'estar a la **mateixa xarxa** — mateix rang, mateix gateway.

#### 3. Comprovar la connectivitat

1. Des del Ubuntu, fes `ping` a la IP del Windows.
2. Des del Windows, fes `ping` a la IP del Ubuntu.
3. Des de tots dos, fes `ping www.google.com` per validar sortida a Internet i DNS.

### Verificació

Les dues VMs han rebut IP dins del rang de la LAN del laboratori, es veuen entre elles i tenen sortida a Internet. El pont ha funcionat: la VM és, a efectes pràctics, un dispositiu més de la LAN.

## Part 2 — Configuració del punt d'accés wifi D-Link

### Objectius

- Reiniciar l'AP D-Link a valors de fàbrica.
- Configurar-hi la interfície LAN amb IP fixa.
- Activar i configurar el **servidor DHCP integrat** de l'AP.

### Tasques

#### 1. Reiniciar l'AP a valors de fàbrica

1. Consulta el manual del D-Link (o la etiqueta física) per veure com fer un **reset de fàbrica** (típicament, prémer el botó de reset amb un clip mentre s'engega, uns 10 segons).
2. Un cop reiniciat, connecta't per **cable Ethernet** des del teu PC a un dels ports LAN de l'AP.
3. Configura el teu PC amb una IP del rang per defecte del D-Link (o deixa DHCP si el punt d'accés en porta un actiu de fàbrica).
4. Obre el navegador i entra a l'adreça d'administració web (normalment `http://192.168.0.1` o similar; comprova el manual).
5. Fes login amb l'usuari/contrasenya per defecte.

#### 2. Configurar la interfície LAN

1. Al menú **LAN Setup** (o similar), assigna a l'AP una IP fixa: `192.168.100.1/24`.
2. Desa i espera que l'AP es reiniciï. Comprovarà que ara l'admin està a `http://192.168.100.1`.
3. Torna a configurar el teu PC perquè agafi IP per DHCP i comprova que rebi una IP del nou rang.

#### 3. Activar el servidor DHCP integrat

Al menú **DHCP Server** (o similar):

- **Enable DHCP Server**: activat.
- **Rang d'IPs**: `192.168.100.50` – `192.168.100.150`.
- **Subnet Mask**: `255.255.255.0`.
- **Gateway**: `192.168.100.1` (el mateix AP).
- **Primary DNS**: `8.8.8.8`.
- **Secondary DNS**: `1.1.1.1`.
- **Lease Time**: 1 hora (3600 segons).

Desa la configuració.

#### 4. Configurar el wifi

Al menú **Wireless**:

- **SSID**: `Practica_RA7_grupN` (substitueix N pel teu grup).
- **Seguretat**: WPA2-PSK (o WPA3-PSK si el D-Link ho suporta).
- **Contrasenya**: la que trieu; comuniqueu-la al grup.

Desa.

### Verificació

L'AP està emetent un SSID visible, i el servidor DHCP integrat està preparat per assignar IPs del rang `192.168.100.50–150`.

## Part 3 — Connexió de les VMs via bridge cap al wifi

### Objectius

- Canviar l'adaptador pont de les dues VMs perquè faci pont sobre l'**adaptador wifi** del host.
- Comprovar que les VMs reben IP del **DHCP integrat de l'AP** (rang 192.168.100.50–150).
- Consultar la **taula de clients DHCP** de l'AP i verificar que hi apareixen les VMs.

### Tasques

#### 1. Connectar el host a l'AP per wifi

1. Al **host**, desconnecta el cable Ethernet (o desactiva l'adaptador Ethernet).
2. Connecta't al SSID `Practica_RA7_grupN` amb la contrasenya configurada.
3. Comprova que el host ha rebut una IP dins de `192.168.100.50–150` (`ip a` o `ipconfig`).

#### 2. Reconfigurar el pont de les VMs cap al wifi

1. Atura les dues VMs.
2. Torna a **Configuració → Xarxa → Adaptador 1** de cadascuna:
   - **Attached to**: `Bridged Adapter`.
   - **Name**: aquesta vegada, l'**adaptador wifi** del host (per exemple `wlp3s0`, `wlan0`, ...).
3. Desa i arrenca-les.

#### 3. Verificar la IP rebuda a les VMs

1. Al **Ubuntu** VM:

   ```bash
   ip a
   ip route
   resolvectl status
   ping 192.168.100.1
   ping www.google.com
   ```

2. Al **Windows** VM:

   ```
   ipconfig /all
   ping 192.168.100.1
   ping www.google.com
   ```

3. Confirma que:
   - La IP està dins del rang `192.168.100.50–150`.
   - El gateway és `192.168.100.1` (l'AP).
   - Els DNS són `8.8.8.8` i `1.1.1.1`.

#### 4. Ping entre VMs

Fes ping entre l'Ubuntu VM i el Windows VM. Ha de funcionar: totes dues estan a la mateixa xarxa `192.168.100.0/24`, ara servida per l'AP wifi.

#### 5. Consultar la taula de clients de l'AP

1. Des del host, torna a la interfície d'administració de l'AP (`http://192.168.100.1`).
2. Ves a **Status → DHCP Clients** (o similar).
3. Identifica les entrades corresponents a:
   - Ubuntu VM (per la seva MAC).
   - Windows VM (per la seva MAC).
   - El propi host.
4. Anota el **lease time** restant per a cada un.
5. Reinicia una de les VMs i comprova com es renova el lease.

### Preguntes de reflexió (per anotar al dossier)

- Quina diferència hi ha, des del punt de vista del client, entre rebre IP per un cable de coure o per una xarxa wifi?
- Quina diferència hi ha entre el DHCP integrat de l'AP i el servei `isc-dhcp-server` que veurem a RA1? Fixa't sobretot en la seva **finalitat** i el seu **rol** dins d'una infraestructura més gran.
- Què passaria si es connectés un altre servidor DHCP a la mateixa xarxa? (**Pista**: DHCP no té autenticació.)
- Per què hem hagut de canviar l'adaptador físic del pont entre la Part 1 i la Part 3?
