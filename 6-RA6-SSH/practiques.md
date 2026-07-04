# Pràctiques RA6 — SSH i accés remot

> **Entorn**: dues màquines virtuals a la mateixa xarxa **host-only** de VirtualBox (per exemple, `192.168.56.0/24`).
>
> - **Servidor**: Ubuntu Server (`192.168.56.10` en els exemples).
> - **Client**: Ubuntu Desktop (`192.168.56.20`).
>
> El **client Windows** és **opcional a les parts 1 i 2** — Windows 10+ ja porta client OpenSSH incorporat, i qui vulgui pot fer-hi les proves. **A la Part 3 sí que cal** un client Windows per fer les combinacions RDP↔Linux i VNC ho podem restringir al parell Linux↔Linux.
>
> Al llarg de totes les pràctiques, cada vegada que modifiquem `/etc/ssh/sshd_config` seguirem aquest ritual:
>
> 1. Fer còpia de seguretat: `sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`.
> 2. Editar el fitxer.
> 3. Validar la sintaxi: `sudo sshd -t` (silenci = tot correcte).
> 4. Reiniciar el servei: `sudo systemctl restart ssh`.
> 5. **No tancar la sessió actual** fins comprovar amb una **segona sessió nova** que la configuració funciona.

## Part 1 — SSH bàsic, `scp` i X11 forwarding

### Objectius

- Instal·lar i posar en marxa un servidor OpenSSH.
- Connectar-se al servidor des d'un client Linux.
- Entendre el missatge de primera connexió i el fitxer `known_hosts`.
- Copiar fitxers entre màquines amb `scp`.
- Executar aplicacions gràfiques remotes amb X11 forwarding.

### Tasques

#### 1. Instal·lació i comprovació

1. Al **servidor**, actualitza els repositoris i instal·la el paquet `openssh-server`.
2. Comprova que el servei està actiu: `systemctl status ssh`.
3. Anota la IP del servidor amb `ip a`.
4. Al **client**, comprova que ja hi ha el paquet `openssh-client` instal·lat (normalment sí, per defecte).

#### 2. Primera connexió i `known_hosts`

1. Des del client, connecta't al servidor amb `ssh usuari@IP_SERVIDOR`.
2. Observa el missatge d'advertència de "authenticity of host can't be established". Contesta `yes`.
3. Un cop dins, mira el contingut de `~/.ssh/known_hosts` al client. Hi ha d'aparèixer una línia amb la clau pública del servidor.
4. Fes `exit` per sortir. Torna a connectar-te — aquesta vegada **no** ha d'aparèixer cap advertència.

#### 3. Simulació d'un canvi de clau (opcional però molt didàctic)

1. Al **servidor**, com a root, esborra i regenera les claus de host:

   ```bash
   sudo rm /etc/ssh/ssh_host_*
   sudo dpkg-reconfigure openssh-server
   sudo systemctl restart ssh
   ```

2. Torna a connectar des del client. Ara ha d'aparèixer el missatge en majúscules **`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`**. Fes una **captura de pantalla** al dossier.
3. Explica al dossier per què apareix aquest missatge i què significaria si aparegués sense haver reinstal·lat el servidor.
4. Neteja l'entrada antiga: `ssh-keygen -R IP_SERVIDOR`. Torna a connectar i acceptar la nova clau.

#### 4. Còpia de fitxers amb `scp`

1. Al client, crea un fitxer `prova.txt` amb qualsevol contingut.
2. Copia'l al servidor a `/home/usuari/`:

   ```bash
   scp prova.txt usuari@IP_SERVIDOR:/home/usuari/
   ```

3. Comprova que ha arribat entrant per SSH i fent `ls`.
4. Al servidor, crea el fitxer `/etc/hostname_copia.txt` amb el contingut de `/etc/hostname`. Baixa'l al directori actual del client:

   ```bash
   scp usuari@IP_SERVIDOR:/etc/hostname_copia.txt .
   ```

5. Copia tot un directori sencer del client al servidor amb `scp -r`.

#### 5. Execució remota d'aplicacions gràfiques (X11 forwarding)

1. Al **servidor** (que no té entorn gràfic), instal·la una aplicació gràfica lleugera:

   ```bash
   sudo apt install xeyes
   ```

   *(Alternativa: `gedit`, `gimp`, `nautilus`, ... el que vulgueu.)*

2. Comprova que a `/etc/ssh/sshd_config` la línia `X11Forwarding yes` **està activa** (per defecte ho està, però ho verifiquem).
3. Des del **client** (Linux Desktop), connecta't amb l'opció `-X`:

   ```bash
   ssh -X usuari@IP_SERVIDOR
   ```

4. Un cop dins, executa `xeyes`. Ha d'aparèixer la finestra a la **teva** pantalla, tot i que el procés corre al **servidor**. Fes una captura.

### Verificació

- La connexió SSH funciona sense advertències després de la primera acceptació.
- `scp` copia fitxers en les dues direccions.
- Una aplicació gràfica del servidor es dibuixa a la pantalla del client via `-X`.

### Reflexió

- Per què el fitxer `known_hosts` és, en la pràctica, "una CA manual"?
- Quina diferència hi ha entre `scp` i copiar per una carpeta compartida SMB/NFS?

## Part 2 — Autenticació per clau, securització i túnels

### Objectius

- Generar un parell de claus i configurar l'accés SSH sense contrasenya.
- Deshabilitar l'accés amb contrasenya al servidor.
- Aplicar les 4 mesures mínimes de securització (`PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 3`, `AllowUsers`).
- Instal·lar i configurar UFW i fail2ban.
- Crear túnels SSH locals per accedir a serveis interns del servidor.

### Tasques

#### 1. Generació de claus

1. Al **client**, genera un parell de claus ED25519:

   ```bash
   ssh-keygen -t ed25519
   ```

2. Accepta el camí per defecte (`~/.ssh/id_ed25519`). Posa una **passphrase** (recomanat) o deixa-la en blanc per aquesta pràctica.
3. Comprova el que s'ha generat: `ls -l ~/.ssh/`. Has de veure `id_ed25519` (privada) i `id_ed25519.pub` (pública).

#### 2. Còpia de la clau pública al servidor

1. Copia la clau pública al servidor amb `ssh-copy-id`:

   ```bash
   ssh-copy-id usuari@IP_SERVIDOR
   ```

2. Comprova al servidor que la clau ha aparegut a `~/.ssh/authorized_keys` de l'usuari:

   ```bash
   cat ~/.ssh/authorized_keys
   ```

3. Ara prova de tornar a connectar: `ssh usuari@IP_SERVIDOR`. Ja no ha de demanar contrasenya (només la passphrase de la clau, si n'has posat).

#### 3. Còpia manual de la clau (per si un dia no tenim `ssh-copy-id`)

1. Al client, mostra el contingut de la clau pública: `cat ~/.ssh/id_ed25519.pub`.
2. Al servidor, edita el fitxer `~/.ssh/authorized_keys` a mà i afegeix una nova línia amb aquesta clau. Vigila permisos: `chmod 700 ~/.ssh` i `chmod 600 ~/.ssh/authorized_keys`.
3. Explica al dossier per què `ssh-copy-id` és més segur que fer-ho a mà.

#### 4. Securització del servidor

Editem `/etc/ssh/sshd_config` per aplicar les 4 mesures mínimes. Recorda el ritual (còpia de seguretat, editar, `sudo sshd -t`, reiniciar, comprovar amb sessió nova).

1. Descomenta o afegeix les línies següents:

   ```
   PermitRootLogin no
   PasswordAuthentication no
   MaxAuthTries 3
   AllowUsers usuari
   ```

   (Substitueix `usuari` pel teu nom d'usuari real al servidor.)

2. Reinicia el servei i **des d'una segona sessió** verifica:
   - Que continues podent entrar amb la clau.
   - Que `ssh root@IP_SERVIDOR` **no** deixa entrar (missatge de "Permission denied").
   - Que si intentes forçar contrasenya amb `ssh -o PubkeyAuthentication=no usuari@IP_SERVIDOR` **no** funciona.

#### 5. Tallafoc (UFW)

1. Al servidor, instal·la UFW si no hi és: `sudo apt install ufw`.
2. Permet només el port SSH:

   ```bash
   sudo ufw allow ssh
   sudo ufw enable
   ```

3. Comprova l'estat: `sudo ufw status`.
4. Prova des del client que la connexió SSH continua funcionant.

#### 6. Protecció contra força bruta amb fail2ban

1. Al servidor, instal·la fail2ban: `sudo apt install fail2ban`.
2. Comprova l'estat del filtre SSH:

   ```bash
   sudo fail2ban-client status sshd
   ```

3. Des del client, provoca uns quants intents fallits (per exemple, connecta't com a un usuari inexistent unes 5-6 vegades):

   ```bash
   ssh usuariinexistent@IP_SERVIDOR
   ```

4. Torna a executar `sudo fail2ban-client status sshd` al servidor i observa si la IP del client apareix a la llista de bloquejats.
5. Per desbloquejar-te: `sudo fail2ban-client set sshd unbanip IP_CLIENT`.

#### 7. Túnel SSH local

Volem accedir a un servei que només escolta a `localhost` del servidor —típicament una base de dades o un panell d'administració intern. Simulem-ho amb un servidor web mínim.

1. Al servidor, instal·la un web server ràpid i posa'l a escoltar només a localhost:

   ```bash
   sudo apt install python3
   sudo python3 -m http.server 8080 --bind 127.0.0.1
   ```

   Deixa la comanda oberta a una terminal del servidor.

2. Comprova al servidor que el servei respon: `curl http://127.0.0.1:8080`.
3. Comprova al **client** que **NO** hi ha resposta si intenta directament: `curl http://IP_SERVIDOR:8080` (ha de fallar — el servei no escolta a la IP externa).
4. Ara, des del client, obre un túnel SSH:

   ```bash
   ssh -L 9090:localhost:8080 usuari@IP_SERVIDOR
   ```

5. Deixa aquesta sessió oberta. A una altra terminal del client, prova `curl http://localhost:9090`. Ara sí que arriba, a través del túnel.
6. Explica al dossier què està passant, dibuixant l'esquema client ↔ servidor amb els ports 9090 (client) i 8080 (servidor).

### Verificació

- L'entrada per contrasenya al servidor està desactivada; només s'entra amb clau.
- `root` no pot entrar directament.
- Només l'usuari llistat a `AllowUsers` pot entrar.
- Intents fallits repetits queden bloquejats per fail2ban.
- Un servei intern al servidor és accessible des del client a través d'un túnel `-L`.

### Reflexió

- Per què *canviar el port SSH* NO és una mesura de seguretat real?
- Quins són els riscos d'un `PasswordAuthentication no` mal fet (com et pots quedar bloquejat fora)?
- En quins casos reals de la vida professional creus que faries servir un túnel `-L`?

## Part 3 — Escriptori remot: VNC i RDP

### Objectius

- Configurar `xrdp` al servidor per rebre connexions RDP des de Windows.
- Utilitzar Remmina al Linux Desktop per controlar un Windows via RDP.
- Muntar un servidor VNC en un Linux i encapsular la connexió dins un túnel SSH.

### Entorn addicional per aquesta part

Cal disposar de:

- El **Servidor Ubuntu** de les parts anteriors (però ara amb un entorn gràfic mínim instal·lat — XFCE, per lleuger).
- El **Client Ubuntu Desktop**.
- Un **Client Windows** (Windows 10 o 11 Pro; l'edició Home fa de client RDP però no de servidor).

### Tasques

#### 1. Preparar entorn gràfic mínim al servidor

`xrdp` necessita un entorn d'escriptori instal·lat al servidor. Farem servir XFCE per la seva lleugeresa.

1. Al servidor, instal·la XFCE:

   ```bash
   sudo apt install xfce4 xfce4-goodies
   ```

2. Instal·la `xrdp`:

   ```bash
   sudo apt install xrdp
   ```

3. Fes que `xrdp` fagi servir XFCE. Al fitxer `~/.xsession` del teu usuari:

   ```bash
   echo "xfce4-session" > ~/.xsession
   ```

4. Reinicia `xrdp`:

   ```bash
   sudo systemctl restart xrdp
   ```

5. Obre el port 3389 al tallafoc: `sudo ufw allow 3389/tcp`.

#### 2. Windows → Linux (Remote Desktop Connection → `xrdp`)

1. Al client Windows, obre "Conexión a Escritorio remoto" (`mstsc`).
2. Escriu la IP del servidor Ubuntu i connecta.
3. Introdueix l'usuari i contrasenya del servidor. Ha d'aparèixer un escriptori XFCE.
4. Fes una **captura de pantalla** amb l'escriptori del servidor visible al Windows.
5. Tanca la sessió correctament (menú de XFCE → Log Out).

#### 3. Linux → Windows (Remmina → RDP)

1. Al client Windows, ves a **Configuració → Sistema → Escritorio remoto** i **habilita** l'escriptori remot. (Cal Windows Pro o superior.)
2. Comprova la IP del Windows amb `ipconfig`.
3. Al **client Linux Desktop**, obre **Remmina** (`sudo apt install remmina remmina-plugin-rdp` si no hi és).
4. Crea una nova connexió:
   - **Protocol**: RDP
   - **Server**: IP del Windows
   - **User name** / **Password**: les del Windows
   - Marca "Use client resolution" per a pantalla completa.
5. Connecta. Ha d'aparèixer l'escriptori de Windows.
6. Captura de pantalla al dossier.

#### 4. Linux → Linux amb VNC

**Objectiu**: connectar-nos des del client Linux a l'escriptori del servidor Linux via VNC, però **encapsulant el trànsit dins un túnel SSH**, tal com es fa a producció.

1. Al **servidor**, instal·la `tigervnc-standalone-server`:

   ```bash
   sudo apt install tigervnc-standalone-server
   ```

2. Configura la contrasenya VNC:

   ```bash
   vncpasswd
   ```

3. Inicia una sessió VNC escoltant **només a localhost** (per seguretat):

   ```bash
   vncserver -localhost yes :1
   ```

   Això crea una sessió VNC al port 5901 accessible només des de dins del servidor.

4. Verifica que està en marxa: `ss -tlnp | grep 5901` — ha d'escoltar només a `127.0.0.1:5901`.

5. Al **client**, obre un túnel SSH cap al port VNC:

   ```bash
   ssh -L 5901:localhost:5901 usuari@IP_SERVIDOR
   ```

6. En una altra terminal del client, obre el visor VNC:

   ```bash
   sudo apt install remmina-plugin-vnc
   remmina -c vnc://localhost:5901
   ```

   O bé directament amb `xtigervncviewer localhost:5901`.

7. Introdueix la contrasenya VNC. Ha d'aparèixer un escriptori del servidor.
8. Captura al dossier.

Per acabar, atura el servidor VNC al servidor:

```bash
vncserver -kill :1
```

### Verificació

- Windows → Linux amb `xrdp` funciona (escriptori XFCE del servidor visible al Windows).
- Linux → Windows amb Remmina funciona (escriptori de Windows visible al Linux).
- Linux → Linux amb VNC funciona, i el trànsit **passa per dins d'un túnel SSH** (podeu comprovar-ho amb `sudo ss -tnp | grep 5901` — la connexió entrant al servidor ve de `127.0.0.1`, no de la IP del client).

### Reflexió

- Per què és tan mala idea exposar directament el port 5900/5901 a Internet?
- Quan triaries RDP i quan VNC?
- Què aporta el túnel SSH davant la contrasenya de VNC?
