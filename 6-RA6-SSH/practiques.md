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

1. Al **servidor**, actualitza els repositoris i instal·la el paquet `openssh-server` si no ertà ja instal·lat.
2. Comprova que el servei està actiu: `systemctl status ssh`.
3. Anota la IP del servidor amb `ip a`.
4. Al **client**, comprova que ja hi ha el paquet `openssh-client` instal·lat (normalment sí, per defecte).

#### 2. Primera connexió i `known_hosts`

1. Des del client, connecta't al servidor amb `ssh usuari@IP_SERVIDOR`.
2. Observa el missatge "authenticity of host can't be established" i el tipus de clau presentat. Encara no contestis `yes`.
3. Obre directament la consola del servidor a VirtualBox. Si el client presenta una clau ED25519, executa:

   ```bash
   ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
   ```

   Si presenta un altre tipus de clau, consulta el fitxer públic corresponent.

4. Compara el valor `SHA256:...` amb el que mostra el client. Si coincideix, contesta `yes`; si no coincideix, atura't i investiga la diferència.
5. Introdueix la contrasenya del compte remot quan es demani. Acceptar la clau del servidor i autenticar-te al compte són passos diferents.
6. Un cop dins, consulta `~/.ssh/known_hosts` **a l'ordinador client**, en un altre terminal. Hi ha d'haver una entrada amb la clau pública del servidor. El nom o la IP poden aparèixer ocults mitjançant un hash.
7. Fes `exit` per sortir de la sessió remota. Torna a connectar-te: no ha de tornar a preguntar si confies en la clau. La contrasenya del compte remot sí que es continuarà demanant mentre no configuris l'accés amb clau pública.

#### 3. Simulació d'un canvi de clau (opcional però molt didàctic)

1. Al **servidor**, com a root, esborra i regenera les claus de host:

   ```bash
   sudo rm /etc/ssh/ssh_host_*
   sudo dpkg-reconfigure openssh-server
   sudo systemctl restart ssh
   ```

2. Torna a connectar des del client. Ara ha d'aparèixer el missatge en majúscules **`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`**. Fes una **captura de pantalla** al dossier.
3. Explica al dossier per què apareix aquest missatge i què significaria si aparegués sense haver reinstal·lat el servidor.
4. Com que acabes de regenerar les claus expressament, coneixes la causa del canvi. Consulta la nova empremta des de la consola de VirtualBox.
5. Al client, retira l'entrada antiga amb `ssh-keygen -R IP_SERVIDOR`. Torna a connectar-te, comprova que l'empremta coincideix amb la consultada i accepta la nova clau.

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
   sudo apt install x11-apps xauth
   ```

   El paquet `x11-apps` inclou `xeyes`; `xauth` permet gestionar l'autenticació de les connexions X11. Per començar, utilitzarem `xeyes`, que és una aplicació petita.

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

- Què guarda `known_hosts` i què comprova el client amb aquesta informació?
- Quina diferència hi ha entre `scp` i copiar per una carpeta compartida SMB/NFS?

## Part 2 — Autenticació per clau, securització i túnels

### Objectius

- Generar un parell de claus i configurar l'accés SSH sense contrasenya.
- Deshabilitar l'accés amb contrasenya al servidor.
- Aplicar quatre mesures de securització: impedir l'entrada directa de root, exigir clau pública, limitar els intents per connexió i restringir els comptes autoritzats.
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
3. Quins errors ajuda a evitar `ssh-copy-id`? Explica per què una còpia manual ben feta també és vàlida.

#### 4. Securització del servidor

Configurarem quatre mesures: impedir l'entrada directa de root, exigir clau pública, limitar els intents per connexió i restringir els comptes autoritzats. **Abans de començar, comprova que ja pots entrar amb la clau i mantén oberta la sessió actual.**

1. Fes la còpia de seguretat del ritual inicial i revisa `/etc/ssh/sshd_config`. A Ubuntu acostuma a incloure, al principi, aquesta línia:

   ```text
   Include /etc/ssh/sshd_config.d/*.conf
   ```

   Significa que també es llegeixen els fitxers `.conf` d'aquesta carpeta. Per a la majoria de directives, OpenSSH conserva **el primer valor que troba**: una opció definida en un fitxer inclòs abans pot prevaler sobre una línia posterior del fitxer principal. Pots consultar la carpeta amb `sudo ls -l /etc/ssh/sshd_config.d/` i llegir-ne els fitxers amb `sudo less NOM_DEL_FITXER`.

2. Edita amb `sudo` `/etc/ssh/sshd_config` i ajusta les directives següents. Si ja estan actives, modifica-les en lloc d'afegir-les duplicades:

   ```text
   PermitRootLogin no
   PasswordAuthentication no
   PubkeyAuthentication yes
   AuthenticationMethods publickey
   MaxAuthTries 3
   AllowUsers usuari
   ```

   Substitueix `usuari` pel nom del compte remot al qual vols accedir.

   - `PasswordAuthentication no` desactiva el mètode `password`, però un altre mètode, `keyboard-interactive`, també podria demanar una contrasenya.
   - `PubkeyAuthentication yes` permet utilitzar clau pública.
   - `AuthenticationMethods publickey` **exigeix** aquest mètode: activar-lo com a possibilitat i exigir-lo són coses diferents.

3. Comprova la sintaxi abans de reiniciar:

   ```bash
   sudo sshd -t
   ```

   Si no mostra cap missatge, no s'han detectat errors de sintaxi. Si mostra errors, corregeix-los abans de continuar.

4. Comprova també els valors efectius que el servidor interpretaria en llegir els fitxers:

   ```bash
   sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication|authenticationmethods|maxauthtries|allowusers) '
   ```

   Han de correspondre a les sis directives del pas 2; OpenSSH mostra els noms en minúscules. Si un valor no coincideix, revisa si ja està definit abans al fitxer principal o en un dels fitxers inclosos. Fes una còpia de seguretat del fitxer que hagis de modificar, corregeix la definició que preval i repeteix `sshd -t` i `sshd -T`.

   **`sshd -t` comprova que la configuració està ben escrita; `sshd -T` mostra els valors que se n'obtenen.** Aquestes ordres no apliquen els canvis al servei que ja està funcionant. Per a aquesta pràctica utilitzem directives globals, fora de blocs `Match`, que serveixen per aplicar opcions només a determinades connexions.

5. Quan les comprovacions siguin correctes, reinicia amb `sudo systemctl restart ssh` i, **sense tancar la sessió actual**, obre una connexió nova des d'un altre terminal del client. Verifica:
   - Que continues podent entrar amb la clau.
   - Que `ssh root@IP_SERVIDOR` **no** deixa entrar (missatge de "Permission denied").
   - Que si desactives l'ús de claus al client amb `ssh -o PubkeyAuthentication=no usuari@IP_SERVIDOR`, **no** pots entrar amb contrasenya ni amb un altre mètode.

   Les comprovacions dels fitxers no substitueixen aquesta prova amb una connexió nova.

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

Fail2ban llegeix els registres del servei, compta els errors que reconeix i bloqueja temporalment la IP d'origen quan se supera el límit configurat. Una *jail* és el conjunt de regles de detecció i bloqueig d'un servei.

1. Al **servidor**, instal·la Fail2ban i el suport per llegir el registre de systemd:

   ```bash
   sudo apt install fail2ban python3-systemd
   ```

2. Edita amb `sudo` el fitxer `/etc/fail2ban/jail.d/sshd.local` i configura la protecció d'SSH. Si ja conté un apartat `[sshd]`, modifica'l en lloc de duplicar-lo:

   ```ini
   [sshd]
   enabled = true
   backend = systemd
   maxretry = 3
   findtime = 10m
   bantime = 2m
   ```

   - `enabled`: activa la protecció del servei SSH.
   - `backend = systemd`: llegeix el diari de systemd, on es registren els esdeveniments d'SSH.
   - `maxretry = 3`: bloqueja una IP quan detecta tres errors reconeguts pel filtre dins de l'interval indicat.
   - `findtime = 10m`: busca aquests errors dins dels darrers deu minuts.
   - `bantime = 2m`: manté el bloqueig durant dos minuts, un temps curt per al laboratori.

3. Valida la configuració, reinicia Fail2ban i comprova la protecció d'SSH. Si la validació dona un error, corregeix-lo abans de reiniciar:

   ```bash
   sudo fail2ban-client -t
   sudo systemctl restart fail2ban
   sudo fail2ban-client status sshd
   ```

4. Deixa oberta la **consola del servidor a VirtualBox**. Des del client, intenta connectar-te repetidament amb un usuari que no existeixi, sense crear-lo al servidor:

   ```bash
   ssh -o BatchMode=yes usuariinexistent@IP_SERVIDOR
   ```

   `BatchMode=yes` evita les preguntes interactives. Observa els comptadors amb `sudo fail2ban-client status sshd` des de la consola del servidor i repeteix la prova fins que s'apliqui el bloqueig. Comptem errors de registre reconeguts pel filtre, no necessàriament ordres `ssh` executades.

5. Comprova que la IP del client apareix a `Banned IP list` i que una connexió SSH nova des d'aquell client queda bloquejada. Si els comptadors no augmenten, consulta `sudo journalctl -u ssh --since "10 minutes ago"` i `sudo journalctl -u fail2ban --since "10 minutes ago"` per comprovar els registres i detectar errors.
6. **Des de la consola de VirtualBox**, desbloqueja el client:

   ```bash
   sudo fail2ban-client set sshd unbanip IP_CLIENT
   ```

   Substitueix `IP_CLIENT` per la IP real del client. Comprova que pots obrir una connexió SSH nova abans de continuar amb els túnels.

#### 7. Túnel SSH local

Volem accedir a un servei que només escolta a `localhost` del servidor —típicament una base de dades o un panell d'administració intern. Simulem-ho amb un servidor web mínim.

1. Al **servidor**, prepara un directori amb una pàgina de prova i inicia un servidor web que només escolti a `127.0.0.1`:

   ```bash
   sudo apt install python3 curl
   mkdir -p ~/web-prova
   echo 'Servei intern accessible pel tunel SSH' > ~/web-prova/index.html
   python3 -m http.server 8080 --bind 127.0.0.1 --directory ~/web-prova
   ```

   Deixa la comanda oberta a una terminal del servidor. El servidor web no necessita `sudo` per escoltar al port 8080 i només comparteix el directori de prova.

2. En una altra terminal del **servidor**, comprova que el servei respon amb `curl http://127.0.0.1:8080`. Després executa `ss -ltn 'sport = :8080'`: l'adreça local ha de ser `127.0.0.1:8080`, no `0.0.0.0:8080` ni la IP de la xarxa del laboratori.
3. Al **client**, instal·la `curl` si no hi és (`sudo apt install curl`) i prova `curl --connect-timeout 3 http://IP_SERVIDOR:8080`. Ha de fallar. **Aquesta fallada, tota sola, no demostra on escolta el servei**, perquè el tallafoc també pot impedir-hi l'accés. La comprovació amb `ss` del pas anterior és la que ens mostra l'adreça d'escolta.
4. Ara, des del client, obre un túnel SSH:

   ```bash
   ssh -N -L 127.0.0.1:9090:127.0.0.1:8080 usuari@IP_SERVIDOR
   ```

5. Deixa aquesta sessió oberta. `-N` manté el túnel sense obrir una consola remota. A una altra terminal del **client**, prova `curl http://127.0.0.1:9090`. Ara ha d'aparèixer el contingut de la pàgina de prova a través del túnel.
6. Explica al dossier què està passant, dibuixant l'esquema client ↔ servidor amb els ports 9090 (client) i 8080 (servidor).
7. Tanca el túnel amb `Ctrl+C` i repeteix el `curl` al port 9090: ara ha de fallar. Atura també el servidor web amb `Ctrl+C` a la terminal on l'has iniciat.

### Verificació

- L'entrada per contrasenya al servidor està desactivada; només s'entra amb clau.
- `root` no pot entrar directament.
- Només l'usuari llistat a `AllowUsers` pot entrar.
- Intents fallits repetits queden bloquejats per fail2ban.
- Un servei intern al servidor és accessible des del client a través d'un túnel `-L`.

### Reflexió

- Què aporta canviar el port SSH i quins problemes de seguretat no resol?
- Què has de comprovar abans d'exigir l'accés només amb clau pública per no quedar-te fora del servidor?
- Quina diferència hi ha entre `sshd -t`, `sshd -T` i provar una connexió nova?
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

1. Al **servidor**, instal·la TigerVNC i el suport de D-Bus per a la sessió gràfica. XFCE ja ha d'estar instal·lat des de l'apartat anterior:

   ```bash
   sudo apt install tigervnc-standalone-server tigervnc-tools dbus-x11
   ```

2. Amb el **compte remot que utilitzarà l'escriptori**, sense `sudo`, configura la contrasenya VNC:

   ```bash
   tigervncpasswd
   ```

   Aquesta contrasenya és pròpia de VNC: no és la del compte remot ni la frase de pas de la clau SSH. Si pregunta si vols una contrasenya de només visualització, contesta que no per a aquesta pràctica.

3. Comprova que existeix `/usr/share/xsessions/xfce.desktop`. El nom `xfce` identifica la sessió gràfica que volem iniciar. Si no existeix, revisa la instal·lació de XFCE abans de continuar.
4. Inicia TigerVNC amb la sessió **XFCE explícita** i escoltant **només a localhost**:

   ```bash
   tigervncserver -localhost yes :1 -- xfce
   ```

   `:1` és el número de pantalla virtual; per defecte, el port és `5900 + 1 = 5901`. `-- xfce` selecciona l'escriptori que s'executarà dins d'aquesta pantalla virtual. És una sessió gràfica nova, no necessàriament la mateixa que veuríem a la consola física.

5. Verifica la sessió amb `tigervncserver -list` i executa `ss -ltn 'sport = :5901'`. Ha d'escoltar només a les adreces de bucle local (`127.0.0.1` i, si apareix, `::1`), no a totes les interfícies. Si la sessió es tanca immediatament, consulta el registre que indica TigerVNC en arrencar.

6. Al **client**, obre un túnel SSH cap al port VNC:

   ```bash
   ssh -N -L 127.0.0.1:5901:127.0.0.1:5901 usuari@IP_SERVIDOR
   ```

7. Deixa el túnel obert. En una altra terminal del **client**, instal·la i obre el visor VNC:

   ```bash
   sudo apt install remmina remmina-plugin-vnc
   remmina -c vnc://127.0.0.1:5901
   ```

   Com a alternativa, instal·la `tigervnc-viewer` al client i executa `xtigervncviewer 127.0.0.1::5901`. En aquest visor, els dos punts dobles indiquen un port TCP explícit.

8. Introdueix la contrasenya VNC. Ha d'aparèixer l'escriptori XFCE de la sessió virtual del servidor.
9. Captura al dossier.

Per acabar, atura el servidor VNC al servidor:

```bash
tigervncserver -kill :1
```

Tanca també el visor i el túnel SSH amb `Ctrl+C` a la terminal del client on l'has iniciat.

### Verificació

- Windows → Linux amb `xrdp` funciona (escriptori XFCE del servidor visible al Windows).
- Linux → Windows amb Remmina funciona (escriptori de Windows visible al Linux).
- Linux → Linux amb VNC funciona, i el trànsit **passa per dins d'un túnel SSH**. Durant la connexió, comproveu al servidor amb `ss -tn 'sport = :5901'` que la connexió VNC ve de `127.0.0.1`. La connexió que travessa la xarxa entre les dues màquines és la d'SSH.

### Reflexió

- Per què és tan mala idea exposar directament el port 5900/5901 a Internet?
- Quan triaries RDP i quan VNC?
- Què aporta el túnel SSH davant la contrasenya de VNC?
