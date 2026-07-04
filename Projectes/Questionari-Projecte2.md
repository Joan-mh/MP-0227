# Qüestionari — Projecte 2 (Proxmox + Guacamole)

> Cada membre del grup l'ha d'entregar **individualment**. Els grups de Ruta A no fan aquest qüestionari.
>
> Nom: _______________________  Grup: ____  Data: __________

## Preguntes

1. Què és **Proxmox VE** i quina diferència pràctica té respecte a fer servir VirtualBox al vostre portàtil? Per què s'instal·la com a **sistema base** de la màquina i no com una aplicació més?

2. Dins de Proxmox, quines **3 VMs** heu creat i quins recursos els heu assignat (RAM, disc, CPU)? Digues per què heu triat aquests valors.

3. Explica com heu configurat la **xarxa interna** de Proxmox perquè les 3 VMs es puguin veure entre elles. Nomeneu la `vmbr` que heu fet servir.

4. Què és **Apache Guacamole** i com aconsegueix mostrar l'escriptori d'una VM Windows dins d'un navegador web sense instal·lar cap plugin ni client al PC de l'usuari?

5. Quin **mètode d'instal·lació** de Guacamole heu triat (Docker o compilació manual) i per què? Si va ser manual, digues on us vau quedar més encallats.

6. Escriu la configuració de la connexió **RDP** que heu creat a Guacamole per la VM Windows: quins paràmetres li heu posat (hostname, port, usuari, contrasenya)?

7. Com heu preparat la **VM Ubuntu Desktop** per rebre connexions per RDP o VNC des de Guacamole? Digueu quin paquet vau instal·lar i quines línies de configuració heu tocat.

8. Guacamole té un sistema d'usuaris i permisos: expliqueu com heu fet que `usuari1` vegi **només** la VM Windows i `usuari2` vegi **només** la VM Ubuntu Desktop.

9. Al RA6 vau muntar RDP i VNC directament a través del client Remmina. Ara ho feu a través de Guacamole. **Quin protocol utilitza el navegador de l'usuari final per parlar amb Guacamole**? (Pista: no és RDP ni VNC; és el que fa el "miracle" de portar-ho a HTML5.)

10. Ha aparegut cap **problema seriós** durant el projecte que hagueu hagut de resoldre? Explica'l amb una o dues frases i com el vau superar. Si no n'ha aparegut cap, digues que tot ha anat rodat.
