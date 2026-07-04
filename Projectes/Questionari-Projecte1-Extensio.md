# Qüestionari — Projecte 1, extensió Ruta A (Squid + Virtualmin)

> Cada membre del grup l'ha d'entregar **individualment**. Els grups de Ruta B (Projecte 2) no fan aquest qüestionari.
>
> Nom: _______________________  Grup: ____  Data: __________

## Preguntes sobre Squid

1. Quina diferència pràctica hi ha entre el **proxy directe** que heu muntat amb Squid i el proxy invers que ja feu servir amb Nginx? Explica-ho amb una frase, indicant on està cada un dins de l'esquema.

2. Escriu el bloc d'ACLs i `http_access` amb què bloquegeu els **3 dominis d'oci** que heu escollit. Digues com heu comprovat des d'un client intern que la restricció funciona.

3. Com heu bloquejat la descàrrega d'**executables** (`.exe`, `.msi`, `.bat`)? Escriu l'ACL i explica per què heu triat `urlpath_regex` en comptes de `dstdomain`.

4. La restricció horària (`08:00-15:00`, dl-dv) l'heu comprovada canviant l'hora del sistema. Explica **com** heu fet la prova i què heu vist al `access.log`.

5. Escriu la línia del `squid.conf` que us garanteix que **cap petició no llistada** en cap ACL personalitzada pot navegar.

## Preguntes sobre Virtualmin

6. Explica per què hem instal·lat **Virtualmin** en aquesta ruta i què aporta respecte al muntatge manual del Projecte 1: quines feines ara les fa el panel enlloc de fer-les vosaltres.

7. Quins passos ha de fer un **client final** (per exemple `client1.<domini>.edu`) des del panel per crear-se un compte de correu i pujar la seva web? Descriu-ho amb 3-5 passos numerats, tal com ho ha fet la persona a qui heu fet la demo al vídeo.

8. Virtualmin toca fitxers de configuració de **Postfix, Dovecot, Apache/Nginx, ProFTPD, BIND**. Digueu almenys un fitxer que Virtualmin modifica **per compte propi** i què hi afegeix (per exemple, un nou virtual host, un mailbox, una zona DNS).

9. Al panel del client final, no li surten totes les opcions que veu l'administrador. Quines diferències has notat entre el **panel d'administrador** i el **panel d'usuari**?

10. Ha aparegut algun **conflicte** entre la configuració manual que ja teníeu al Projecte 1 i la que Virtualmin volia imposar? Explica'l amb una o dues frases i com el vau resoldre (o si l'heu esquivat instal·lant Virtualmin en un servidor fresc).
