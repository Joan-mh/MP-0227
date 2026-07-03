# Pràctiques RA4 — Correu (Postfix + Dovecot)

> **Xarxa**: `192.168.X.0/24`, el tercer octet `X` el trieu vosaltres. El **domini** l'inventeu (per exemple `cognom.edu`).
>
> El **servidor** Ubuntu és, alhora, el DNS (de la Part 2 endavant) i el Postfix/Dovecot. Als exemples: `mail = 192.168.50.20`.
>
> Client de correu recomanat: **Evolution** (`sudo apt install evolution`). Podeu provar Thunderbird, però probablement rebutjarà el certificat autofirmat de la Part 3 — llavors passeu a Evolution o valideu amb `openssl s_client`.

## Part 1 — Clients de correu amb comptes reals (2 hores)

### Objectius

- Instal·lar i configurar un client de correu (Thunderbird, opcionalment també a Windows).
- Entendre les diferències pràctiques entre **POP3** i **IMAP**.
- Comprovar que el correu funciona entre proveïdors diferents (Gmail, Outlook, Yahoo).

### Preparació

Necessiteu **dos comptes de correu de proveïdors diferents**. Recomanacions:

- Un compte a Gmail (o Outlook).
- Un compte a Yahoo (o similar).

> **Consell**: si feu servir un compte que ja teniu en actiu, no configureu POP3 sobre ell perquè es descarregarà tot el correu al client i el pot esborrar del servidor. **Feu comptes nous per fer proves** o assegureu-vos que el client està en mode "deixar còpies al servidor".

Per als proveïdors amb doble autenticació (Gmail, Outlook), sovint cal generar una **contrasenya d'aplicació** — el compte principal no accepta que un client extern hi accedeixi amb la contrasenya normal.

### Tasques

#### 1. Instal·lació de Thunderbird

Al client Linux (i opcionalment també a Windows):

```bash
sudo apt install thunderbird
```

#### 2. Compte 1 amb POP3

1. Obriu Thunderbird i configureu el **compte 1** (per exemple, Gmail) amb **POP3** de rebuda i **SMTP** d'enviament.
2. Anoteu els servidors i ports que us proposa el client.
3. Envieu un correu al compte 2 i comproveu que arriba.
4. Al servidor (via web), comproveu si el correu ha desaparegut de la bústia (POP3 típicament el descarrega i esborra) o si s'ha mantingut (depèn de la configuració fina de POP3).

#### 3. Compte 2 amb IMAP

1. Configureu el **compte 2** (per exemple, Yahoo o Outlook) amb **IMAP** de rebuda i **SMTP** d'enviament.
2. Envieu un correu del compte 2 al compte 1 i comproveu que arriba.
3. Al servidor (via web), comproveu que el correu **hi és** després de llegir-lo (IMAP no esborra).

#### 4. Comparativa

Contesta al vostre dossier:

- Quines diferències de configuració heu vist entre POP3 i IMAP?
- Què fa el servidor **SMTP**? Per què necessiteu configurar-lo per als dos comptes independentment del protocol de rebuda?
- En quin context faríeu servir POP3 avui dia? I IMAP?

Adjunteu **captures de pantalla** de les dues configuracions i de la prova d'enviament creuada.

### Resultats esperats

- Els dos comptes envien i reben correu correctament amb els protocols configurats.
- Documenteu les diferències observades entre POP3 i IMAP.

---

## Part 2 — Servidor Postfix + Dovecot amb DNS propi (3 hores)

### Objectius

- Preparar el DNS (registre MX + A) al servidor bind9 de RA2.
- Instal·lar i configurar **Postfix** com a MTA.
- Instal·lar i configurar **Dovecot** com a MDA amb IMAP i POP3.
- Enviar correu entre usuaris **locals** amb la comanda `mail`.
- Configurar Evolution al client per llegir les bústies **sense TLS** encara.

### Preparació

Al servidor cal tenir Ubuntu Server amb el rol de DNS ja configurat (RA2). Assumim `mail.cognom.edu = 192.168.50.20` amb un únic servidor.

### Tasques

#### 1. Preparació del DNS

Al fitxer de zona directa (`/etc/bind/db.cognom.edu`) afegeix:

- Un registre **A** per a `mail` amb la IP del vostre servidor.
- Un registre **MX** apuntant a `mail.cognom.edu.` amb preferència `10`.

**Incrementa el serial del SOA**, valida amb `named-checkzone` i reinicia bind9.

Verifica-ho des del client:

```bash
dig cognom.edu MX
dig mail.cognom.edu
```

#### 2. Configurar hostname i /etc/hosts

Al servidor:

```bash
sudo hostnamectl set-hostname mail.cognom.edu
```

I ajusta `/etc/hosts` perquè el nom del servidor resolgui a la seva IP:

```
127.0.0.1        localhost
192.168.50.20    mail.cognom.edu mail
```

#### 3. Instal·lar Postfix

```bash
sudo apt update
sudo apt install postfix mailutils
```

L'assistent apareixerà automàticament. Trieu les 9 opcions segons la teoria (Internet Site, nom del domini, xarxes locals amb la vostra xarxa, límit `0`, protocol `all`).

> **Recordatori**: `sudo dpkg-reconfigure postfix` es pot tornar a executar en qualsevol moment. Fes una còpia de `main.cf` abans.

#### 4. Ajustar `main.cf` per Maildir

```bash
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.bck
```

Edita `/etc/postfix/main.cf` i afegeix o modifica:

```
home_mailbox = Maildir/
```

Comprova que aquestes línies també hi són (ho ha de deixar l'assistent):

```
myhostname = mail.cognom.edu
mydestination = $myhostname, cognom.edu, localhost.cognom.edu, localhost
inet_interfaces = all
inet_protocols = all
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.50.0/24
```

Reinicia:

```bash
sudo systemctl restart postfix
```

#### 5. Instal·lar Dovecot

```bash
sudo apt install dovecot-imapd dovecot-pop3d
```

Configura:

- **`/etc/dovecot/conf.d/10-mail.conf`** — `mail_location = maildir:~/Maildir`.
- **`/etc/dovecot/conf.d/10-auth.conf`** — `disable_plaintext_auth = no` i `auth_mechanisms = plain login`.
- **`/etc/dovecot/dovecot.conf`** — afegir `login_trusted_networks = 192.168.50.0/24`.

Reinicia:

```bash
sudo systemctl restart dovecot
```

#### 6. Crear usuaris i enviar correu local

```bash
sudo adduser correu1
sudo adduser correu2
```

Envia un correu amb `mail`:

```bash
echo "Prova des de correu1" | mail -s "Hola" correu2@cognom.edu
```

Comprova que ha arribat entrant com a `correu2` i executant `mail`. Explora les ordres de dins de `mail` (`h`, `n`, `p`, `d`, `q`).

#### 7. Comprovar amb telnet

Des del **client** (o des del propi servidor):

**POP3 al port 110**:

```bash
telnet 192.168.50.20 110
USER correu2
PASS <contrasenya>
STAT
LIST
RETR 1
QUIT
```

**IMAP al port 143**:

```bash
telnet 192.168.50.20 143
a LOGIN correu2 <contrasenya>
a LIST "" "*"
a SELECT INBOX
a FETCH 1 BODY[]
a LOGOUT
```

Els dos han de mostrar el missatge que has enviat abans.

#### 8. Configurar Evolution al client

Instal·la Evolution al client Linux:

```bash
sudo apt install evolution
```

Configura **dos comptes**, un per a `correu1` i un altre per a `correu2`:

- **Correu 1**: rebuda **POP3** al port 110, enviament SMTP al port 25.
- **Correu 2**: rebuda **IMAP** al port 143, enviament SMTP al port 25.

⚠️ En aquesta fase, **sense TLS**. Cal marcar "no seguretat" a la configuració (Dovecot ja té `disable_plaintext_auth = no` i la vostra IP està a `login_trusted_networks`).

Envia correus entre els dos comptes des d'Evolution i comprova que arriben.

### Resultats esperats

- El DNS propi resol el registre MX correctament (`dig cognom.edu MX`).
- La comanda `mail` envia i rep correu local.
- Telnet mostra que POP3 i IMAP funcionen.
- Evolution envia i rep correu entre `correu1@cognom.edu` i `correu2@cognom.edu` sense xifratge.

---

## Part 3 — Servidor de correu segur amb TLS (2 hores) — GUIADA

> ⚠️ **Aquesta pràctica és guiada des del principi**. Us donem tots els fitxers de configuració llestos. La vostra tasca és **executar els passos i mostrar que el resultat funciona** — no heu de deduir res. Feu-la amb calma llegint què fa cada canvi.

### Objectius

- Generar un certificat autofirmat amb OpenSSL.
- Activar **SMTPS** (port 465) al Postfix.
- Activar **IMAPS** (port 993) al Dovecot.
- Verificar el xifratge sense dependre d'un client (amb `openssl s_client`).
- Configurar Evolution per als ports segurs.

### Punt de partida

La Part 2 ha de funcionar completament. Recomanem fer una **captura del `main.cf` actual** abans de començar (per si cal recuperar-lo):

```bash
sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.bck-part3
sudo cp /etc/postfix/master.cf /etc/postfix/master.cf.bck-part3
sudo cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.bck-part3
```

### Pas 1 — Generar el certificat autofirmat

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/mail.key \
    -out /etc/ssl/certs/mail.crt \
    -subj "/CN=mail.cognom.edu" \
    -addext "subjectAltName=DNS:mail.cognom.edu"
sudo chmod 600 /etc/ssl/private/mail.key
```

**Comprovació**:

```bash
sudo ls -l /etc/ssl/private/mail.key /etc/ssl/certs/mail.crt
sudo openssl x509 -in /etc/ssl/certs/mail.crt -noout -text | head -20
```

### Pas 2 — Configurar Postfix per SMTPS al port 465

Edita `/etc/postfix/main.cf` i afegeix (o modifica) al final:

```
smtpd_tls_cert_file=/etc/ssl/certs/mail.crt
smtpd_tls_key_file=/etc/ssl/private/mail.key
smtpd_use_tls=yes
smtpd_tls_security_level=may
smtp_tls_security_level=may

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
```

Edita `/etc/postfix/master.cf` i **descomenta** (o afegeix) el bloc de l'`smtps` al principi del fitxer:

```
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

> ℹ️ Les línies que comencen amb `-o` són **opcions específiques** d'aquest servei. Cal que estiguin **indentades amb espai** (no tabulador).

Reinicia:

```bash
sudo systemctl restart postfix
```

### Pas 3 — Configurar Dovecot per IMAPS al port 993

Crea o edita `/etc/dovecot/conf.d/10-ssl.conf` i posa-hi:

```
ssl = required
ssl_cert = </etc/ssl/certs/mail.crt
ssl_key  = </etc/ssl/private/mail.key
```

> ℹ️ El `<` de davant de la ruta és intencional: indica a Dovecot que ha de **llegir el fitxer** i incloure'n el contingut.

A `/etc/dovecot/conf.d/10-master.conf`, afegeix el bloc de comunicació amb Postfix (per SASL):

```
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

Reinicia:

```bash
sudo systemctl restart dovecot
```

### Pas 4 — Verificar els ports

Al servidor:

```bash
sudo ss -tlnp | grep -E ':(25|465|993|110|143)'
```

Han d'aparèixer, com a mínim, els ports **25** (SMTP), **465** (SMTPS), **993** (IMAPS) i **143** (IMAP). Els altres poden estar-hi o no depenent de si has deshabilitat els no xifrats.

### Pas 5 — Verificar amb `openssl s_client` (sense client de correu)

Aquesta és **la manera més robusta** de comprovar que TLS funciona: no depèn de cap client i mostra el certificat directament.

**IMAPS al port 993**:

```bash
openssl s_client -connect mail.cognom.edu:993
```

Ha de mostrar el certificat i acabar amb un prompt IMAP. Prova a fer:

```
a LOGIN correu1 <contrasenya>
a LIST "" "*"
a LOGOUT
```

**SMTPS al port 465**:

```bash
openssl s_client -connect mail.cognom.edu:465
```

Un cop dins escriu:

```
EHLO localhost
QUIT
```

Ha de respondre amb un `250 Hello` seguit de les extensions suportades (`AUTH PLAIN LOGIN`, `STARTTLS`...).

**Fes captura de pantalla** de les dues comprovacions.

### Pas 6 — Configurar Evolution per als ports segurs

Al client Linux, obre Evolution i crea (o edita) un compte per a `correu1@cognom.edu`:

**Recepció (IMAP)**:

- Servidor: `mail.cognom.edu`.
- Port: **993**.
- Mètode de seguretat: **SSL/TLS al port dedicat**.
- Autenticació: Contrasenya (PLAIN).
- Usuari: `correu1`.

**Enviament (SMTP)**:

- Servidor: `mail.cognom.edu`.
- Port: **465**.
- Mètode de seguretat: **SSL/TLS al port dedicat**.
- Autenticació: Contrasenya (PLAIN).
- Usuari: `correu1`.

Quan Evolution es queixi del **certificat autofirmat** (perquè no ve d'una CA reconeguda), **accepta'l manualment** — Evolution ho permet.

Envia un correu de `correu1@cognom.edu` a `correu2@cognom.edu` i comprova que arriba.

### Pas 7 — Si intentes fer-ho amb Thunderbird

Prova (si vols) de configurar el mateix compte amb Thunderbird. Notaràs que **les versions recents rebutgen el certificat autofirmat** i no permeten acceptar-lo manualment. Documenta l'error que apareix.

Aquesta és la raó per la qual recomanem Evolution.

### Entrega

- Captura de pantalla del `openssl s_client -connect mail.cognom.edu:993` mostrant el certificat.
- Captura de pantalla del `openssl s_client -connect mail.cognom.edu:465` amb el `EHLO`.
- Captura de pantalla d'Evolution enviant i rebent un correu amb el cadenat de seguretat.
- Sortida de `sudo ss -tlnp | grep -E ':(25|465|993)'` mostrant els ports actius.
- Una nota breu (2-3 línies) explicant per què el vostre certificat és autofirmat i què faríeu diferent en producció.

### Resultats esperats

- Postfix escolta al **465** amb TLS wrapping.
- Dovecot escolta al **993** amb TLS.
- El certificat és el vostre autofirmat, amb `CN=mail.cognom.edu`.
- `openssl s_client` valida les dues connexions.
- Evolution envia i rep correu xifrat.
