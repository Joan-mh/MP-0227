# Serveis de xarxa — Correu electrònic (Postfix + Dovecot)

## Introducció històrica

El correu electrònic és **més antic que Internet**. El primer correu es va enviar el **1962** des del MIT i el 1965 ja existia el servei `Mail` per enviar missatges entre terminals connectats.

Quan va arribar ARPANET (1969), la utilitat es va disparar. El **1971** Ray Tomlinson va enviar el primer correu per la xarxa i va introduir la convenció crucial de separar l'usuari de la màquina amb una **`@`** (en anglès es llegeix "at", per això `joan@terranova.com` es llegeix "joan at terranova.com").

Ràpidament van aparèixer proveïdors grans (Hotmail — abans de Microsoft —, Yahoo, Gmail...) i el correu es va convertir en la primera aplicació d'Internet de masses. A l'estat espanyol el primer correu és del **1985**, però no s'estén com a eina d'ús comú fins a principis dels **2000**.

## Arquitectura del correu: MUA / MTA / MDA

El correu no funciona amb una sola màquina ni un sol protocol. Hi ha tres tipus d'actors:

| Actor | Nom complet | Què fa | Exemples |
|---|---|---|---|
| **MUA** | *Mail User Agent* | Client. El programa amb què l'usuari escriu, envia i llegeix correu. | Thunderbird, Evolution, Outlook, `mail` |
| **MTA** | *Mail Transfer Agent* | Servidor. Enruta el correu entre servidors, com un carter. | **Postfix**, Sendmail, QMail, EXIM |
| **MDA** | *Mail Delivery Agent* | Lliurament final: entrega el correu a la bústia de l'usuari i el serveix quan aquest es connecta. | **Dovecot**, procmail |

**Dovecot** és MDA + servidor IMAP/POP3 alhora (dona accés a les bústies).

### Flux típic d'un correu

Suposem que `joan@empresa1.cat` escriu a `maria@empresa2.cat`:

1. **Joan** escriu el missatge al seu MUA (Thunderbird) i li dona a "Enviar".
2. Thunderbird envia el missatge al **MTA d'empresa1** (`smtp.empresa1.cat`) via **SMTP**.
3. L'MTA d'empresa1 consulta el **DNS** buscant el registre **MX** d'`empresa2.cat` per saber a quin servidor entregar.
4. L'MTA d'empresa1 obre una connexió SMTP amb l'MTA d'empresa2 i li lliura el missatge.
5. L'MTA d'empresa2 (Postfix) rep el missatge i el passa al **MDA** (Dovecot), que el deixa a la bústia de `maria`.
6. **Maria** obre el seu MUA, que es connecta al servidor IMAP/POP3 d'empresa2 (Dovecot) i descarrega el missatge.

## Protocols i ports

| Protocol | Rol | Port sense TLS | Port amb TLS |
|---|---|---|---|
| **SMTP** | Enviar correu (client → MTA i MTA → MTA) | 25 (MTA↔MTA) / 587 (client, STARTTLS) | 465 (SMTPS, TLS des del principi) |
| **POP3** | Descarregar la bústia (típicament esborrant-la del servidor) | 110 | 995 (POP3S) |
| **IMAP** | Sincronitzar la bústia (deixa els correus al servidor) | 143 | 993 (IMAPS) |

### Diferència pràctica POP3 vs. IMAP

- **POP3**: descarrega el correu del servidor al dispositiu i (per defecte) l'esborra del servidor. Bo si només consultes des d'un dispositiu i vols estalviar espai al servidor. **Problema**: si obres correu al portàtil i al mòbil, no veus el mateix.
- **IMAP**: manté el correu al servidor i sincronitza els canvis (llegits, carpetes, esborrats) entre tots els dispositius. **Estàndard modern**: tots els proveïdors grans (Gmail, Outlook) el fan servir per defecte.

En qualsevol dels dos casos, l'**enviament** es fa sempre amb **SMTP**.

## STARTTLS vs. TLS "directe" (SMTPS/IMAPS)

Hi ha dues maneres d'afegir xifratge a un protocol antic com SMTP o IMAP:

- **STARTTLS**: la connexió comença sense xifrar al port habitual (25/587 per SMTP, 143 per IMAP). El client envia una comanda `STARTTLS` i, a partir d'aquí, la sessió es xifra.
- **TLS directe** (SMTPS, IMAPS, POP3S): la connexió ja s'obre xifrada des del primer byte, en un **port diferent** (465, 993, 995).

Els proveïdors moderns (Gmail, Outlook) admeten totes dues. En aquest curs farem servir **TLS directe (SMTPS al 465, IMAPS al 993)** perquè és més simple d'entendre.

## Registre MX al DNS (repàs breu)

Un servidor de correu no es descobreix per nom sinó per **registre MX**. Al fitxer de zona del RA2:

```
empresa1.cat.    IN MX 10  mail.empresa1.cat.
mail             IN A      192.168.50.20
```

Interpretació: qui vulgui enviar correu a `qualsevol@empresa1.cat` ha de consultar el registre MX i acabarà entregant al servidor `mail.empresa1.cat` (`192.168.50.20`).

El número **`10`** és la **preferència**: com més baix, més prioritari. Si hi ha diversos MX es proven per ordre de preferència.

Comprovació:

```bash
dig empresa1.cat MX
dig mail.empresa1.cat
```

## Postfix

**Postfix** és l'MTA que farem servir. És el més utilitzat a servidors Linux moderns (majoritari a Ubuntu, Debian, RedHat). Comparat amb els seus germans:

- **Sendmail**: el clàssic. Molt potent però amb sintaxi complexa i quasi mai s'usa a servidors nous.
- **EXIM**: per defecte a Debian pur (no Ubuntu). Sintaxi radicalment diferent, més flexible per a configuracions molt específiques.
- **Postfix**: escrit als 90s a IBM Research per substituir Sendmail. Modular, segur, sintaxi clara. **És el que farem servir tant a pràctiques com al Projecte 1**.

### Instal·lació

```bash
sudo apt update
sudo apt install postfix mailutils
```

L'`apt` obrirà automàticament l'**assistent de configuració** (`dpkg-reconfigure`). Podeu tornar-lo a llançar en qualsevol moment amb:

```bash
sudo dpkg-reconfigure postfix
```

> ⚠️ **Avís important**: `dpkg-reconfigure postfix` **reescriu `/etc/postfix/main.cf`**. Si has fet canvis manuals a aquest fitxer i tornes a executar l'assistent, es perden. Fes una còpia (`sudo cp /etc/postfix/main.cf /etc/postfix/main.cf.bck`) abans d'executar-lo.

### Les 9 preguntes de l'assistent

L'assistent us presentarà 9 pantalles. Aquesta secció explica què triar a cadascuna.

#### 1. Tipus general de configuració de correu

Defineix el **rol** del servidor:

- **Sense configuració**: no fa res.
- **Lloc d'Internet (Internet Site)**: envia i rep directament amb SMTP. Servidor complet i autònom.
- **Internet amb smarthost**: rep directament, però tot el correu que **surt** passa per un altre servidor (el *smarthost*). Útil si el proveïdor bloqueja el port 25.
- **Sistema satèl·lit**: **tot** el correu (entrant i sortint) passa per un altre servidor.
- **Només local**: correu només entre usuaris de la mateixa màquina.

> **Escollir**: **Lloc d'Internet**.

#### 2. Nom de correu del sistema (System mail name)

El **domini** amb què el servidor s'identificarà. S'afegirà a les adreces sense domini explícit.

> **Escollir**: el vostre domini, per exemple `jmercade.com` (el mateix del DNS).

#### 3. Destinatari del correu de root i postmaster

Els correus de sistema (avisos, alertes) es solen redirigir a un usuari administrador.

> **Escollir**: **deixar-ho en blanc** per ara. Aniran a `root`.

#### 4. Altres destinacions per acceptar correu

Els dominis pels quals Postfix acceptarà correu.

> **Escollir**: assegureu-vos que el vostre domini hi apareix. Per defecte sol quedar `$myhostname, jmercade.com, localhost.localdomain, localhost`.

#### 5. Forçar actualitzacions síncrones a la cua de correu

Afecta el rendiment i la integritat.

> **Escollir**: **No** (millor rendiment, l'estàndard).

#### 6. Xarxes locals

Les IPs que Postfix considera **de confiança** per fer *relay*. Per defecte només `127.0.0.0/8` (només la mateixa màquina).

> **Escollir**: afegir la vostra xarxa. Per exemple: `127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.50.0/24`.

#### 7. Límit de mida de la bústia

`0` = sense límit. Recomanat.

> **Escollir**: **`0`**.

#### 8. Caràcter d'extensió d'adreça local

Amb `+` com a extensió, `correu1+notificacions@jmercade.com` s'entrega a `correu1`. Molt útil per filtrar.

> **Escollir**: `+` (per defecte).

#### 9. Protocols d'Internet a utilitzar

- **all**: IPv4 i IPv6.
- **ipv4** / **ipv6**: només un dels dos.

> **Escollir**: **all**.

### Fitxers principals de Postfix

Tot dins `/etc/postfix/`:

- **`main.cf`** — configuració principal (paràmetres, TLS, restriccions).
- **`master.cf`** — quins serveis s'activen i com (SMTP al 25, submission al 587, smtps al 465...).
- **`/etc/mailname`** — nom de correu del sistema (el que va donar al pas 2 de l'assistent).

### Format de bústia: Maildir vs. mbox

- **`mbox`** (per defecte): tots els correus d'una bústia en **un únic fitxer** (`/var/mail/<usuari>`). Simple però lent i propens a corrupció si molts processos hi escriuen alhora.
- **`Maildir`** (recomanat modern): **un fitxer per correu** dins d'una estructura de directoris (`~/Maildir/{new,cur,tmp}`). Robust, ràpid, sense bloqueigs.

Postfix per defecte usa `mbox`. Per canviar-lo a `Maildir` a `main.cf`:

```
home_mailbox = Maildir/
```

Cal reiniciar el servei. **Dovecot ha d'usar el mateix format** (veure secció Dovecot).

## Dovecot

**Dovecot** és el nostre servidor **IMAP + POP3** (i alhora fa d'MDA). Instal·lació:

```bash
sudo apt install dovecot-imapd dovecot-pop3d
```

### Fitxers principals

Tot dins `/etc/dovecot/`:

- **`dovecot.conf`** — punt d'entrada. Inclou tots els altres.
- **`conf.d/10-mail.conf`** — on són les bústies. Ha de coincidir amb Postfix.
- **`conf.d/10-auth.conf`** — mecanismes d'autenticació.
- **`conf.d/10-master.conf`** — serveis: quin protocol escolta a quin port.
- **`conf.d/10-ssl.conf`** — configuració TLS.

### Configuració mínima

**`10-mail.conf`** — coincidir amb el format de Postfix:

```
mail_location = maildir:~/Maildir
```

**`10-auth.conf`** — permetre autenticació sense TLS **NOMÉS per proves**:

```
disable_plaintext_auth = no
auth_mechanisms = plain login
```

En producció, `disable_plaintext_auth = yes` i només amb TLS.

**`dovecot.conf`** — indicar la vostra xarxa local (com a `login_trusted_networks`) per acceptar autenticacions durant la fase de proves sense TLS:

```
login_trusted_networks = 192.168.50.0/24
```

### Comprovar Dovecot per telnet

Els protocols POP3 i IMAP són **de text**: es poden provar amb `telnet` sense necessitat de cap client.

**POP3 (port 110):**

```bash
telnet localhost 110
USER correu1
PASS <la_contrasenya>
STAT
LIST
RETR 1
QUIT
```

**IMAP (port 143):**

```bash
telnet localhost 143
a LOGIN correu1 <contrasenya>
a LIST "" "*"
a SELECT INBOX
a FETCH 1 BODY[]
a LOGOUT
```

L'`a` de davant de cada comanda és una **etiqueta** (qualsevol lletra o número): IMAP en necessita una per identificar la resposta de cada comanda.

## La comanda `mail`

`mail` (paquet `mailutils`) és un MUA de línia d'ordres. No admet fitxers adjunts però és perfecte per a proves ràpides des del terminal.

### Enviar un correu

```bash
echo "Hola, això és una prova" | mail -s "Assumpte" correu1@jmercade.com
```

Modificadors útils:

- `-s "text"` — assumpte.
- `-v` — verbose (útil per veure què fa).
- `-a fitxer` — adjuntar un fitxer (només amb versions modernes de `mailutils`).

Format interactiu:

```
$ mail correu1@jmercade.com
Cc:
Subject: Test
Cos del missatge
línia 2
^D             (Ctrl+D per finalitzar el missatge)
EOT
```

### Llegir la bústia

Un cop tens missatges:

```
$ mail
"/var/mail/joanm": 3 messages 3 new
>N 1 root         Fri Jan 9 05:54  19/817   Missatge 1
 N 2 root         Fri Jan 9 05:56  19/783   Missatge 2
 N 3 root         Fri Jan 9 05:59  20/895   Missatge 3
?
```

El símbol **`>`** marca el missatge actual (sobre el qual actuaran les ordres si no s'especifica).

### Ordres dins de `mail`

| Ordre | Ús |
|---|---|
| `n` (o Enter) | Llegir el següent missatge. |
| `p` | Reimprimir el missatge actual. |
| `h` | Capçaleres de tots els missatges. |
| `d [num]` | Esborrar el missatge actual o el `num`. |
| `r [num]` | Respondre. |
| `q` | Sortir preservant els no llegits. |
| `x` | Sortir preservant tots els missatges (**inclús els esborrats amb `d`** — no confirma!). |
| `list` | Veure totes les ordres disponibles. |

### Estats dels missatges

- **`N`** — nou (no llegit).
- **`R`** — llegit.
- **`U`** — no llegit però conegut (no és nou).
- **`>`** — missatge actual.

## Cifrat: certificats autofirmats i Let's Encrypt

Per xifrar SMTP i IMAP necessitem un **certificat X.509** (com HTTPS). Dues opcions:

### Certificat autofirmat (OpenSSL) — el que farem

Es genera al mateix servidor amb OpenSSL. Vàlid tècnicament, però **no signat per cap autoritat certificadora**: els clients avisaran que "no confien" en el certificat.

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/mail.key \
    -out /etc/ssl/certs/mail.crt \
    -subj "/CN=mail.jmercade.com"
```

Ideal per laboratori i xarxes internes. **Els nostres exercicis usen sempre autofirmats.**

### Let's Encrypt (ACME)

**Let's Encrypt** és una autoritat certificadora gratuïta i automatitzada. Emet certificats signats de veritat (els navegadors i clients hi confien sense avisar). S'obtenen amb l'eina **`certbot`** o similar.

**Per què no la fem servir aquí**: Let's Encrypt necessita **un domini públic real** i que el servidor sigui **accessible des d'Internet** per verificar la propietat del domini (via HTTP-01 o DNS-01 challenge). Als laboratoris del CFGM treballem amb dominis inventats a xarxes host-only, on no arriba Let's Encrypt. En un servidor de producció amb domini real, seria la primera opció.

## Autenticació anti-suplantació moderna: SPF, DKIM, DMARC

Aquests tres registres publiquen al DNS "com identificar" el correu legítim del vostre domini. Els grans proveïdors (Gmail, Outlook) els comproven i marquen com **spam** els correus que no els respectin.

- **SPF** (*Sender Policy Framework*, RFC 7208) — un registre **TXT** al DNS que llista quines IPs poden enviar correu en nom del vostre domini. Exemple:

    ```
    jmercade.com. IN TXT "v=spf1 mx ~all"
    ```

    "El correu legítim ve dels servidors llistats al MX; qualsevol altre (`~all`) marqueu-lo com a sospitós."

- **DKIM** (*DomainKeys Identified Mail*, RFC 6376) — signa criptogràficament els correus sortints amb una clau privada. La clau **pública** es publica al DNS com a TXT. El receptor verifica la firma amb aquella clau, garantint que el correu no s'ha alterat i ve realment de qui diu.

- **DMARC** (*Domain-based Message Authentication, Reporting and Conformance*, RFC 7489) — política que diu al receptor **què fer** amb els correus que fallen SPF o DKIM (acceptar, marcar com a spam, rebutjar) i on enviar els informes.

En aquest RA no els configurarem (requereixen domini públic i coordinació amb el DNS extern), però són **conceptes fonamentals de qualsevol servidor de correu modern** i entren a l'examen.

## Clients de correu: Thunderbird i Evolution

Els dos són MUAs gràfics disponibles a Ubuntu.

- **Thunderbird** — el més popular. Fàcil d'usar, molt configurable.
- **Evolution** — més integrat amb GNOME, funcionalitats de calendari/agenda.

### El problema de Thunderbird amb certificats autofirmats

Les **últimes versions de Thunderbird rebutgen els certificats autofirmats** i, a diferència de com era abans, **no permeten acceptar-los manualment**. Sembla un bug seriós de Thunderbird més que no una decisió deliberada. La versió 0.99 encara funcionava.

Per als exercicis d'aquest RA:

- **Client principal recomanat**: **Evolution** (`sudo apt install evolution`). Accepta certificats autofirmats sense problemes.
- Si voleu comprovar l'autofirmat sense cap client, es pot fer amb `openssl s_client`:

    ```bash
    openssl s_client -connect mail.jmercade.com:993
    openssl s_client -connect mail.jmercade.com:465
    ```

    Aquesta comanda mostra el certificat i permet parlar el protocol manualment: és la forma més neta de validar que el servidor xifra correctament sense dependre de cap client gràfic.

## Referències

- Documentació oficial de Postfix: [postfix.org/documentation.html](https://www.postfix.org/documentation.html)
- Documentació de Dovecot: [doc.dovecot.org](https://doc.dovecot.org/)
- RFC 5321 — SMTP.
- RFC 3501 — IMAP v4.
- RFC 1939 — POP3.
- RFC 7208 — SPF. RFC 6376 — DKIM. RFC 7489 — DMARC.
