# Mòdul 227 — Serveis de Xarxa (CFGM SMX)

Material docent obert del mòdul professional **0227 Serveis de Xarxa** del cicle formatiu de grau mitjà **Sistemes Microinformàtics i Xarxes**. Cobreix els 8 resultats d'aprenentatge oficials del mòdul (132 hores) amb teoria, pràctiques i projectes finals.

- **Idioma**: català
- **Sistema operatiu del servidor**: Ubuntu Server (LTS)
- **Sistema operatiu del client**: Ubuntu Desktop, Windows 10/11 o el que trieu (indicat a cada pràctica)
- **Format**: Markdown pur, pensat per llegir-se directament a GitHub

## Ordre real d'impartició

L'ordre no és el numèric dels RA — està pensat perquè cada mòdul reforci coneixements del previ:

| Ordre | RA | Servei | Pes | Teoria | Pràctiques |
|:-:|:-:|---|:-:|:-:|:-:|
| 1 | RA7 | Wifi (repàs TCP/IP + adaptador pont) | 10% | [teoria](7-RA7-Wifi/teoria.md) | [pràctiques](7-RA7-Wifi/practiques.md) |
| 2 | RA6 | SSH i accés remot | 10% | [teoria](6-RA6-SSH/teoria.md) | [pràctiques](6-RA6-SSH/practiques.md) |
| 3 | RA1 | DHCP | 15% | [teoria](1-RA1-DHCP/teoria.md) | [pràctiques](1-RA1-DHCP/practiques.md) |
| 4 | RA2 | DNS | 15% | [teoria](2-RA2-DNS/teoria.md) | [pràctiques](2-RA2-DNS/practiques.md) |
| 5 | RA3 | FTP | 10% | [teoria](3-RA3-FTP/teoria.md) | [pràctiques](3-RA3-FTP/practiques.md) |
| 6 | RA5 | HTTP | 15% | [teoria](5-RA5-HTTP/teoria.md) | [pràctiques](5-RA5-HTTP/practiques.md) |
| 7 | RA8 | Proxy | 10% | [teoria](8-RA8-Proxy/teoria.md) | [pràctiques](8-RA8-Proxy/practiques.md) |
| 8 | RA4 | Correu (Postfix + Dovecot) | 15% | [teoria](4-RA4-Mail/teoria.md) | [pràctiques](4-RA4-Mail/practiques.md) |

Les carpetes mantenen la numeració oficial dels RA (`1-RA1-DHCP`, `2-RA2-DNS`, ...) independentment de l'ordre real d'impartició.

## Estructura de cada RA

Cada carpeta segueix la mateixa estructura de fitxers públics:

```
N-RAx-Servei/
├── teoria.md          Fonaments teòrics i referències
└── practiques.md       Mínim 3 parts amb dificultat creixent
```

Al RA7 hi ha, a més, un `checklist.md` amb els punts observables en directe (aquest RA no té examen escrit).

## Projectes finals

Al final del mòdul es fa un projecte de 36 hores. Cada grup tria una de les dues rutes:

- **Ruta A — Projecte 1 complet**: DHCP + DNS + FTP + HTTP + Correu (**Postfix + Dovecot amb TLS**) + Squid (proxy directe) + **Virtualmin** (panel d'ISP).
- **Ruta B — Projecte 1 parcial + Projecte 2**: DHCP + DNS + FTP + HTTP + Correu + **Proxmox VE + Apache Guacamole** (escriptori remot des del navegador).

Enllaços:

- [Projectes/Projecte1.md](Projectes/Projecte1.md) — nucli obligatori + secció final "Extensió Ruta A".
- [Projectes/Projecte2.md](Projectes/Projecte2.md) — Proxmox + Guacamole (només Ruta B).
- Qüestionaris (10 preguntes obertes cadascun, per validar que **cada membre** del grup coneix la feina):
    - [Questionari-Projecte1-Base.md](Projectes/Questionari-Projecte1-Base.md) — respon tothom.
    - [Questionari-Projecte1-Extensio.md](Projectes/Questionari-Projecte1-Extensio.md) — només Ruta A.
    - [Questionari-Projecte2.md](Projectes/Questionari-Projecte2.md) — només Ruta B.

## Com fer servir aquest repositori

**Alumnat**: llegiu `teoria.md`, feu `practiques.md` en ordre. Cada pràctica té una secció d'objectius, tasques i verificació. Entregueu els projectes i les pràctiques al Moodle abans de les dates de tancament previstos. No s'admeten treballs fora de termini.

## Ferramenta principal per servei

| Servei               | A pràctiques                   | Al Projecte 1                 | Notes                                          |
| -------------------- | ------------------------------ | ----------------------------- | ---------------------------------------------- |
| DHCP                 | isc-dhcp-server                | Kea DHCP                      | Kea és el successor modern (esmentat a teoria) |
| DNS                  | BIND9                          | BIND9                         |                                                |
| FTP                  | vsftpd                         | ProFTPD                       | Els dos són vàlids; ProFTPD és més modular     |
| HTTP                 | Apache                         | Apache + Nginx (frontal)      | Nginx com a proxy invers i balancejador        |
| Proxy                | Squid                          | Squid + Nginx                 | Squid = directe, Nginx = invers                |
| Correu               | Postfix + Dovecot              | idem                          | TLS obligatori; client recomanat: Evolution    |
| SSH i accés remot    | OpenSSH + Remmina + xrdp + VNC | —                             | Reservat per teoria/pràctica, no al projecte   |
| Wifi                 | Punt d'accés D-Link físic      | —                             | Introducció; ús real més avall                 |
| Escriptori remot web | —                              | Proxmox VE + Apache Guacamole | Només al Projecte 2 (Ruta B)                   |
| Panel d'ISP          | —                              | Virtualmin                    | Només al Projecte 1 extensió (Ruta A)          |

## Llicència

Els materials estan alliberats sota llicència [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/). Podeu utilitzar-los, adaptar-los i distribuir-los, sempre reconeixent l'autoria original i mantenint la mateixa llicència.

## Contribucions

Les propostes de millora són benvingudes: obriu un *issue* o un *pull request*. Manteniu el català i el format Markdown pur (sense numeració manual als encapçalaments, blocs de codi amb llenguatge declarat, taules Markdown netes).
