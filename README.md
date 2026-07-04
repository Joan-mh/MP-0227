# Mòdul 227 — Serveis de Xarxa (CFGM SMX)

Material docent obert del mòdul professional **0227 Serveis de Xarxa** del cicle formatiu de grau mitjà **Sistemes Microinformàtics i Xarxes**. Cobreix els 8 resultats d'aprenentatge oficials del mòdul (132 hores) amb teoria, pràctiques i projectes finals.

- **Idioma**: català
- **Sistema operatiu del servidor**: Ubuntu Server (LTS)
- **Sistema operatiu del client**: Ubuntu Desktop, Windows 10/11 o el que trieu (indicat a cada pràctica)
- **Format**: Markdown pur, pensat per llegir-se directament a GitHub

## Ordre real d'impartició

L'ordre no és el numèric dels RA — està pensat perquè cada mòdul reforci coneixements del previ:

| Ordre | RA | Servei | Pes | Enllaç |
|:-:|:-:|---|:-:|---|
| 1 | RA7 | Wifi (repàs TCP/IP + adaptador pont) | 10% | [7-RA7-Wifi/](7-RA7-Wifi/) |
| 2 | RA6 | SSH i accés remot | 10% | [6-RA6-SSH/](6-RA6-SSH/) |
| 3 | RA1 | DHCP | 15% | [1-RA1-DHCP/](1-RA1-DHCP/) |
| 4 | RA2 | DNS | 15% | [2-RA2-DNS/](2-RA2-DNS/) |
| 5 | RA3 | FTP | 10% | [3-RA3-FTP/](3-RA3-FTP/) |
| 6 | RA5 | HTTP | 15% | [5-RA5-HTTP/](5-RA5-HTTP/) |
| 7 | RA8 | Proxy | 10% | [8-RA8-Proxy/](8-RA8-Proxy/) |
| 8 | RA4 | Correu (Postfix + Dovecot) | 15% | [4-RA4-Mail/](4-RA4-Mail/) |

Les carpetes mantenen la numeració oficial dels RA (`1-RA1-DHCP`, `2-RA2-DNS`, ...) independentment de l'ordre real d'impartició.

## Estructura de cada RA

Cada carpeta segueix la mateixa estructura de fitxers públics:

```
N-RAx-Servei/
├── teoria.md          Fonaments teòrics i referències
└── practiques.md       Mínim 3 parts amb dificultat creixent
```

Al RA7 hi ha, a més, un `checklist.md` amb els punts observables en directe (aquest RA no té examen escrit).

Les solucions de pràctiques i els enunciats/solucions d'exàmens són **material privat del professor** i no formen part del repositori públic (veure [Material privat](#material-privat)).

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

**Alumnat**: llegiu `teoria.md`, feu `practiques.md` en ordre. Cada pràctica té una secció d'objectius, tasques i verificació. Al final del mòdul entregueu els projectes i els qüestionaris.

**Professorat**: podeu fer fork del repositori i adaptar-lo. Les solucions i els exàmens són al vostre `.gitignore` — genereu els vostres propis o poseu-vos en contacte.

## Ferramenta principal per servei

| Servei | A pràctiques | Al Projecte 1 | Notes |
|---|---|---|---|
| DHCP | isc-dhcp-server | isc-dhcp-server | Kea és el successor modern (esmentat a teoria) |
| DNS | BIND9 | BIND9 | |
| FTP | vsftpd | ProFTPD | Els dos són vàlids; ProFTPD és més modular |
| HTTP | Apache | Apache + Nginx (frontal) | Nginx com a proxy invers i balancejador |
| Proxy | Squid | Squid + Nginx | Squid = directe, Nginx = invers |
| Correu | Postfix + Dovecot | idem | TLS obligatori; client recomanat: Evolution |
| SSH i accés remot | OpenSSH + Remmina + xrdp + VNC | — | Reservat per teoria/pràctica, no al projecte |
| Wifi | Punt d'accés D-Link físic | — | Introducció; ús real més avall |
| Escriptori remot web | — | Proxmox VE + Apache Guacamole | Només al Projecte 2 (Ruta B) |
| Panel d'ISP | — | Virtualmin | Només al Projecte 1 extensió (Ruta A) |

## Llicència

Els materials estan alliberats sota llicència [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/). Podeu utilitzar-los, adaptar-los i distribuir-los, sempre reconeixent l'autoria original i mantenint la mateixa llicència.

## Contribucions

Les propostes de millora són benvingudes: obriu un *issue* o un *pull request*. Manteniu el català i el format Markdown pur (sense numeració manual als encapçalaments, blocs de codi amb llenguatge declarat, taules Markdown netes).
