# Checklist d'avaluació RA7 — Wifi

> RA7 s'avalua **en directe durant la pràctica**, no per examen escrit. Aquest full és la referència del professor per posar nota. Cada ítem val els punts indicats; total 10.

**Alumne**: ____________________________  **Grup**: ____  **Data**: __________

---

## Part 1 — Adaptador pont cablejat (3 punts)

- [ ] (0,5) Les dues VMs (Ubuntu + Windows) tenen l'adaptador configurat en **mode pont** cap a l'adaptador Ethernet físic del host.
- [ ] (0,5) Sap identificar quin adaptador físic del host està escollit per fer de pont (`eno1`, `enp3s0`, ...).
- [ ] (1) Totes dues VMs han rebut IP dins del rang de la LAN del laboratori (amb sortida a Internet), i pot mostrar-la amb `ip a` / `ipconfig`.
- [ ] (1) `ping` entre VMs i cap a Internet funcionen; sap interpretar-ne els resultats.

**Subtotal Part 1**: ___ / 3

## Part 2 — Configuració de l'AP D-Link (3 punts)

- [ ] (0,5) L'AP està a `192.168.100.1/24` i s'hi accedeix per navegador.
- [ ] (1,5) El servidor DHCP integrat està activat amb:
  - Rang correcte (`192.168.100.50–150`)
  - Gateway correcte (`192.168.100.1`)
  - DNS correctes (`8.8.8.8`, `1.1.1.1`)
  - Lease de 1 hora
- [ ] (1) El wifi emet amb SSID i seguretat WPA2/WPA3 configurats correctament (contrasenya no trivial).

**Subtotal Part 2**: ___ / 3

## Part 3 — Connexió via wifi i verificació (3 punts)

- [ ] (0,5) Ha canviat l'adaptador pont de les VMs cap a l'adaptador **wifi** del host.
- [ ] (1) Totes dues VMs reben IP dins de `192.168.100.50–150` amb gateway i DNS correctes.
- [ ] (0,5) `ping` entre VMs i cap a Internet funcionen des de wifi.
- [ ] (1) Sap consultar la **taula de clients DHCP** de l'AP, identificar les seves VMs per la MAC i explicar què és el *lease time*.

**Subtotal Part 3**: ___ / 3

## Reflexió i comprensió (1 punt)

- [ ] (0,25) Explica per què wifi i cable són equivalents a nivell lògic.
- [ ] (0,25) Sap diferenciar el DHCP integrat de l'AP del `isc-dhcp-server` que farà a RA1.
- [ ] (0,25) Sap explicar què pot passar amb dos servidors DHCP a la mateixa xarxa (*rogue DHCP*).
- [ ] (0,25) Justifica el canvi d'adaptador físic al pont entre la Part 1 i la Part 3.

**Subtotal reflexió**: ___ / 1

---

## Nota final

**Total**: ___ / 10

**Observacions**:
