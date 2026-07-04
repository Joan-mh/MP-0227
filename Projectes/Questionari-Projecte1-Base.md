# Qüestionari — Projecte 1 (nucli obligatori)

> Cada membre del grup l'ha d'entregar **individualment**. Les respostes són curtes i concretes; l'objectiu és validar que has treballat i que coneixes el projecte, no fer un examen.
>
> Nom: _______________________  Grup: ____  Data: __________

## Preguntes

1. Dibuixa o descriu **amb paraules pròpies** l'esquema de xarxa del vostre projecte. Indica les IPs de les dues interfícies del servidor, la LAN interna, els clients externs i el DNS Root de l'aula.

2. Quin ordre de configuració heu seguit i per què? (per exemple, per què DNS abans que HTTP?).

3. Quina és la funció del **DNS Root** gestionat pel professor i com hi encaixa el vostre servidor DNS?

4. Escriu el fragment de `dhcpd.conf` (o equivalent) que defineix el rang d'IPs que reparteix el vostre servidor DHCP als clients interns. Explica cada línia amb una frase.

5. Quins **registres DNS** (A, MX, PTR...) heu creat per resoldre els vostres dominis, i on estan escrits al fitxer de zona?

6. Explica com heu configurat **FTPS** amb ProFTPD: on està el certificat, quines directives l'activen, i com heu comprovat que funciona des d'un client extern.

7. Com heu aconseguit que **Nginx** faci de proxy invers cap als dos backends Apache? Escriu el bloc `upstream` i el `location` que heu utilitzat, i explica què fa el `weight`.

8. Quina és la diferència pràctica que heu vist entre **HTTPS amb certificat autofirmat** i HTTPS "vàlid" com el de qualsevol web pública? Quin missatge us donen els navegadors?

9. Com heu configurat **Postfix i Dovecot** perquè el correu vagi xifrat amb TLS? Indiqueu almenys dues línies clau de cada fitxer (`main.cf` de Postfix i `10-ssl.conf` o similar de Dovecot).

10. Ha aparegut cap **problema seriós** durant el projecte que hagueu hagut de resoldre? Explica'l amb una o dues frases i com el vau superar. Si no n'ha aparegut cap, digues que tot ha anat rodat.
