# Projecte 2 — Escriptori remot amb Proxmox i Guacamole

> **Aquest projecte només el fan els grups que han triat la Ruta B** (Projecte 1 parcial + Projecte 2). Els grups de Ruta A no fan aquest document; ja han fet Squid + Virtualmin com a extensió del Projecte 1.

**Durada**: aproximadament 15 hores dins del total de 36 h combinades amb el Projecte 1 (parcial).

## Objectius

Crear un prototip d'una plataforma d'aules virtuals — semblant a l'entorn Isard que feu servir al centre, però construït pel vostre grup des de zero:

- Muntar un hipervisor **Proxmox VE** amb diverses màquines virtuals.
- Muntar un servidor **Apache Guacamole** que permeti als usuaris accedir a les seves VMs des d'un navegador web, sense instal·lar cap client.
- Assignar diferents màquines a diferents usuaris.

Aquest projecte reprèn la teoria d'accés remot (RA6: SSH, VNC, RDP) però aplicada dins d'una plataforma d'escriptori remot web.

## Metodologia de treball

Idèntica al Projecte 1:

1. Grups de **2 o 3 persones**.
2. **Portaveu setmanal** per consultes al professor a la seva taula.
3. Entrega: **PDF** amb instruccions + **vídeo** de funcionament.
4. Nota ponderada: 20% autoavaluació + 30% co-avaluació + 50% professor. Diferència de 3 o més punts entre alumne i professor → no compta la de l'alumne.
5. Cada membre del grup entrega individualment el `Questionari-Projecte2.md`.

## Escenari

Topologia bàsica:

```
                          ┌──────────────────────────────────────────┐
                          │       ORDINADOR FÍSIC (Proxmox VE)       │
                          │                                          │
                          │  ┌───────────┐  ┌──────────┐  ┌────────┐ │
                          │  │ VM Guaca- │  │ VM Win-  │  │ VM Ub. │ │
   Usuari                 │  │  mole     │  │  dows 10 │  │ Desktop│ │
   Extern    ── HTTPS ──► │  │ (Ubuntu   │  │  (RDP)   │  │ (xrdp) │ │
   (navega-               │  │  Server)  │  │          │  │        │ │
   dor web)               │  └─────┬─────┘  └────▲─────┘  └───▲────┘ │
                          │        │             │            │      │
                          │        └── RDP/VNC ──┴────────────┘      │
                          │           (vmbr1, xarxa interna)         │
                          └──────────────────────────────────────────┘
```

- **1 ordinador físic** proporcionat pel professor amb capacitat per instal·lar-hi Proxmox VE (no és una VM: Proxmox és el sistema operatiu **base** de la màquina).
- Dins de Proxmox, creareu **almenys 3 màquines virtuals**:
  - **VM Guacamole**: Ubuntu Server on hi correrà el servei Guacamole (frontal web + connectors).
  - **VM Windows 10** (o 11): serveix com a "màquina d'usuari" accessible per RDP.
  - **VM Ubuntu Desktop**: serveix com a "màquina d'usuari" accessible per VNC o RDP (`xrdp`).
- Els usuaris finals accediran a **Guacamole** des del navegador i, un cop autenticats, veuran les VMs que tenen assignades.

## Etapes del projecte

### Fase 1 — Instal·lació de Proxmox VE

- Baixeu la **[ISO oficial de Proxmox VE](https://www.proxmox.com/en/downloads/category/iso-images-pve)**.
- Prepareu un pendrive d'arrencada.
- Instal·leu Proxmox VE a la màquina física. La instal·lació és semblant a la d'un Debian Server.
- Un cop instal·lat, accediu a la interfície web a `https://<IP_Proxmox>:8006`.
- Registreu com a mínim l'usuari `root` i el certificat autofirmat de Proxmox.

### Fase 2 — Crear les VMs a Proxmox

- Pugeu les ISOs necessàries (Ubuntu Server, Ubuntu Desktop, Windows 10/11) a l'**Emmagatzematge → ISO Images**.
- Creeu les 3 VMs assignant recursos mínims (2 GB RAM, 20 GB disc, 1 vCPU per començar).
- Instal·leu els sistemes operatius.
- Configureu la xarxa interna de Proxmox (`vmbr0` o `vmbr1`) perquè totes les VMs es vegin entre elles.

### Fase 3 — Preparar les VMs per accés remot

- A la **VM Windows**: habiliteu **Escriptori remot** (Configuració → Sistema → Escriptori remot → activat). Requereix Windows Pro o superior.
- A la **VM Ubuntu Desktop**: instal·leu `xrdp` (RA6, Part 3), o bé configureu VNC amb `tigervnc-server`. Comproveu que us hi podeu connectar amb un client de la mateixa xarxa (Remmina o similar).

### Fase 4 — Instal·lar Apache Guacamole

Aquí és on hi ha més feina. **Nota**: la [instal·lació moderna de Guacamole](https://guacamole.apache.org/doc/gug/installing-guacamole.html) és més complexa que la que descrivien manuals antics per Ubuntu 16. No proveu de fer servir procediments d'Ubuntu 16.04 sobre Ubuntu Server 22+/24.

Opcions per instal·lar Guacamole:

- **Opció A (recomanada per la vostra primera vegada)** — instal·lació **via Docker**:

    ```bash
    sudo apt install docker.io docker-compose
    ```

    Aleshores feu servir la imatge oficial `guacamole/guacamole` i `guacamole/guacd` amb un `docker-compose.yml`. És, de llarg, la manera més ràpida i menys frustrant.

- **Opció B** — instal·lació **manual** compilant `guacd` i muntant Tomcat + MySQL. És l'opció "aprofundeix", però trigareu 5-8 hores només d'entorn i us podeu quedar encallats.

Si trieu l'opció B i us encalleu, demaneu ajuda al professor: us donarà un procediment funcional, **però amb una petita reducció de nota** (l'aprenentatge era intentar-ho pel vostre compte primer).

### Fase 5 — Configurar connexions a Guacamole

A la interfície web de Guacamole (per defecte `http://<IP_VM_Guacamole>:8080/guacamole`, usuari/contrasenya `guacadmin`/`guacadmin`):

1. Canvieu la contrasenya per defecte.
2. Creeu **connexions** cap a cada VM:
    - Connexió RDP a la VM Windows.
    - Connexió RDP (o VNC) a la VM Ubuntu Desktop.
3. Creeu **usuaris**:
    - Almenys 2 usuaris diferents (per exemple `usuari1`, `usuari2`).
    - Al primer li assigneu **només** la VM Windows.
    - Al segon li assigneu **només** la VM Ubuntu Desktop.
4. Des d'un client extern (un navegador de l'ordinador que useu de treball, no de dins de Proxmox), entreu a Guacamole com a `usuari1` i comproveu que veieu la seva VM. Feu el mateix amb `usuari2`.

### Fase 6 — Documentar i entregar

## Entrega

- **PDF** amb la instal·lació i configuració completa (què heu fet, en quin ordre, quins problemes trobats i com els heu resolt).
- **Vídeo** que mostri:
    - Interfície de Proxmox amb les 3 VMs.
    - Entrada com a `usuari1` a Guacamole des d'un navegador → aparegui només la VM Windows i pugueu controlar-la.
    - Log out i entrada com a `usuari2` → aparegui només la VM Ubuntu Desktop i pugueu controlar-la.
- Cada membre del grup entrega, individualment, `Questionari-Projecte2.md`.

## Recursos

- [Proxmox VE — documentació oficial](https://pve.proxmox.com/pve-docs/)
- [Proxmox VE — instal·lació ISO](https://www.proxmox.com/en/downloads/category/iso-images-pve)
- [Apache Guacamole — Getting Started](https://guacamole.apache.org/doc/gug/getting-started.html)
- [Guacamole amb Docker](https://guacamole.apache.org/doc/gug/guacamole-docker.html)
- Referències dels RA:
    - **RA6** (SSH, VNC, RDP, xrdp) per entendre què transmeten les connexions per sota de Guacamole.
