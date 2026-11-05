# Serveis de Xarxa · RA2 · DNS (Part 1)

## Introducció al DNS i a l'espai de noms

**Cicle:** 2n SMX (Sistemes Microinformàtics i Xarxes)
**Mòdul:** Serveis de Xarxa
**Resultat d'aprenentatge 2:** Instal·lació i configuració de serveis de resolució de noms (DNS)

---

## Índex

1. [Què és el DNS?](#1-què-és-el-dns)
2. [D'on venim: els sistemes de noms plans](#2-don-venim-els-sistemes-de-noms-plans)
3. [L'estructura jeràrquica del DNS](#3-lestructura-jeràrquica-del-dns)
4. [Elements de l'espai de noms](#4-elements-de-lespai-de-noms)
5. [Com s'escriu un nom de domini](#5-com-sescriu-un-nom-de-domini)
6. [FQDN: el nom de domini complet](#6-fqdn-el-nom-de-domini-complet)
7. [Els dominis de primer nivell (TLD)](#7-els-dominis-de-primer-nivell-tld)
8. [Els servidors arrel](#8-els-servidors-arrel)
9. [Avantatges del DNS](#9-avantatges-del-dns)
10. [Comandes útils de Linux](#10-comandes-útils-de-linux)
11. [Glossari](#11-glossari)
12. [Resum](#12-resum)
13. [Preguntes de repàs](#13-preguntes-de-repàs)

---

## 1. Què és el DNS?

### 1.1 Definició

**DNS** són les sigles de **Domain Name System** (*sistema de noms de domini*).

És el servei de xarxa que **tradueix noms de domini a adreces IP** (i també a l'inrevés). Gràcies al DNS podem escriure `www.google.com` al navegador en lloc d'haver de recordar una adreça com `142.250.184.4`.

### 1.2 Una analogia per entendre'l

El DNS funciona com l'**agenda de contactes del mòbil**:

| Agenda del mòbil | DNS |
|---|---|
| Busques **"Mama"** | Escrius **`www.xtec.cat`** |
| L'agenda et dona el **número de telèfon** | El DNS et dona l'**adreça IP** |
| El mòbil truca al número | L'ordinador es connecta a la IP |

Les persones recordem **noms**; els ordinadors treballen amb **números**. El DNS fa de traductor entre tots dos.

### 1.3 Funcions principals del DNS

| Funció | Exemple |
|---|---|
| **Resolució directa:** nom → IP | `www.xtec.cat` → `83.247.151.214` |
| **Resolució inversa:** IP → nom | `8.8.8.8` → `dns.google` |
| **Identificar els servidors de correu** d'un domini | On s'ha d'enviar un correu per a `@gmail.com`? |
| **Identificar els equips (hosts)** de la xarxa | `servidor1.empresa.local` |
| **Identificar altres serveis** | Servidors d'impressió, de veu IP, etc. |

### 1.4 Independència entre el nom i la IP

El DNS separa el **nom** d'un servei de la seva **adreça IP**. Això dona dos grans avantatges:

- **Canvi d'IP transparent:** si un servidor web canvia d'IP, només cal actualitzar el DNS. Els usuaris continuen escrivint el mateix nom i **no se n'adonen**.
- **Redundància:** un mateix nom pot tenir **diverses IP**. Si un servidor cau, els clients poden connectar-se a un altre. Això aporta **alta disponibilitat**.

---

## 2. D'on venim: els sistemes de noms plans

### 2.1 Una mica d'història

Des del principi va caldre un mecanisme en "llenguatge humà" per identificar els equips de la xarxa, sobretot els que oferien serveis.

```text
  ARPANET  ──────▶  hosts.txt  ──────▶  /etc/hosts  ──────▶  DNS
 (anys 70)      (fitxer central)    (encara existeix)   (sistema actual)
```

1. **ARPANET:** la xarxa predecessora d'Internet, creada pel Departament de Defensa dels EUA. Ja assignava noms als equips.
2. **hosts.txt:** els noms es publicaven en **un únic fitxer** que contenia tots els equips i les seves IP. Cada ordinador se'l descarregava periòdicament.
3. **/etc/hosts:** a GNU/Linux encara existeix l'equivalent d'aquell fitxer. A Windows és `C:\Windows\System32\drivers\etc\hosts`.

### 2.2 El sistema de noms pla

Un **sistema de noms pla** és simplement un **fitxer de text** on cada línia relaciona una IP amb un o més noms.

**Característiques:**

- És molt senzill.
- És útil per definir **àlies en xarxes petites** (casa, una aula).
- **No és escalable**: no serveix per a xarxes grans ni per a Internet.

### 2.3 Exemple de fitxer `/etc/hosts`

```text
[root@portatil ~]# cat /etc/hosts
# Do not remove the following line, or various programs
# that require network functionality will fail.
127.0.0.1      localhost.localdomain localhost
::1            localhost6.localdomain6 localhost6
192.168.1.1    router      routerWF
192.168.1.31   server1     escriptori   pare
192.168.1.32   estacio1    dormitori    mare
192.168.1.33   estacio2    nen          jocs     supercrac
```

Format de cada línia: **`IP   nom_principal   àlies1   àlies2 ...`**

> **Fixa't:** l'equip `192.168.1.33` es pot anomenar de **quatre maneres**: `estacio2`, `nen`, `jocs` o `supercrac`. Si fas `ping jocs`, l'ordinador mirarà aquest fitxer i sabrà que ha d'enviar el ping a `192.168.1.33`.

### 2.4 Per què no funciona a Internet?

Imagina fer servir un sol fitxer `hosts` per a tot Internet:

- **Un únic equip** hauria de centralitzar **milers de milions** de noms.
- Cada cop que algú afegís o canviés un nom, s'hauria de **tornar a enviar el fitxer a tots els ordinadors del món**.
- Si aquest equip central fallés, **ningú no podria resoldre noms**.
- Res no impediria que dues persones volguessin el **mateix nom** (col·lisions).

Conclusió: **és inviable**. Calia un sistema **jeràrquic i distribuït** → el **DNS**.

| | **Sistema pla (hosts)** | **Sistema jeràrquic (DNS)** |
|---|---|---|
| **Organització** | Un fitxer centralitzat | Arbre distribuït en molts servidors |
| **Escalabilitat** | No escalable | Escalable fins a tot Internet |
| **Ús típic** | Xarxes petites | Internet i xarxes grans |
| **Noms duplicats** | Possibles | Impossibles (sense col·lisions) |

---

## 3. L'estructura jeràrquica del DNS

El DNS organitza els noms en forma d'**arbre invertit**: l'arrel és a dalt i les branques van cap avall.

```text
                                 .  (arrel)
          ┌──────────┬───────────┼────────────┬──────────┐
         org        net         com          es        info        ← 1r nivell (TLD)
                     │                  ┌─────┼─────┐
                   mired           zeppelinux google  red          ← 2n nivell
        ┌────┬────┬──┴──┬────┐          │      │      │
       ftp  www  smtp  pop  direccion   www    www    www          ← 3r nivell
                              ┌──┴──┐
                            pc01   pc02                            ← 4t nivell
```

- **Domini arrel (`.`):** el punt de dalt de tot. No té nom.
- **Dominis de primer nivell (TLD):** `.com`, `.org`, `.es`, `.cat`...
- **Dominis de segon nivell:** els que registren empreses i organitzacions (`google.es`, `xtec.cat`).
- **Tercer nivell i següents:** subdominis o equips (`www.google.es`, `pc01.direccion.mired.net`).

Aquesta estructura serveix per:

- **Evitar col·lisions** (noms duplicats).
- Poder **identificar qualsevol equip** de la xarxa de manera única.
- **Repartir la feina** entre molts servidors (cadascun s'encarrega d'una part de l'arbre).

---

## 4. Elements de l'espai de noms

| Element | Definició |
|---|---|
| **Espai de noms** | El conjunt de **tots** els dominis que formen l'arbre DNS complet. |
| **Domini** | Un **node** de l'arbre **i tot el que penja d'ell**. Per exemple, el domini `es` inclou `google.es`, `www.google.es`, etc. |
| **Subdomini** | Cada domini es pot dividir en subdominis. `xtec.cat` és subdomini de `cat`, i `jaumehuguet.xtec.cat` és subdomini de `xtec.cat`. |

> **Regla important:** dos nodes **no es poden repetir al mateix nivell d'una mateixa branca**, però **sí en branques diferents**. Per exemple, `www` apareix sota `google`, sota `red` i sota `zeppelinux`, i no passa res perquè són branques diferents.

---

## 5. Com s'escriu un nom de domini

### 5.1 Caràcters permesos

| Norma | Detall |
|---|---|
| **Caràcters** | Lletres llatines (a–z), dígits (0–9) i el guió (`-`). |
| **Majúscules/minúscules** | És **indiferent**: `WWW.XTEC.CAT` = `www.xtec.cat`. |
| **Longitud d'una etiqueta** | Màxim **63 caràcters** per etiqueta (cada tros entre punts). |
| **Longitud total** | Màxim **253 caràcters** per al nom complet. |

> **Nota del professor:** a la presentació hi diu 64 caràcters, però l'estàndard (RFC 1035) fixa un màxim de **63** caràcters per etiqueta.

### 5.2 IDNA: noms amb accents i altres alfabets

**IDNA** (*Internationalized Domain Names in Applications*) és el mecanisme que permet fer servir caràcters d'altres alfabets (accents, ç, ñ, caràcters xinesos, àrabs...).

Internament, el nom es converteix a caràcters bàsics amb un prefix `xn--`. Per exemple:

```text
  cafè.cat   →   xn--caf-dma.cat
```

### 5.3 DNS vs. sistema de fitxers

L'arbre DNS s'assembla a l'arbre de carpetes d'un disc, però **es llegeix al revés**:

| | **Sistema de fitxers** | **DNS** |
|---|---|---|
| **Direcció de lectura** | De l'**arrel** cap al node | Del **node** cap a l'arrel |
| **Separador** | `/` | `.` |
| **Exemple** | `/etc/bind/named.conf` | `ftp.jaumehuguet.xtec.cat.` |
| **On és l'arrel?** | Al **principi** (`/`) | Al **final** (`.`) |

---

## 6. FQDN: el nom de domini complet

### 6.1 Definició

El **FQDN** (*Fully Qualified Domain Name*) és el **nom de domini complet** d'un equip. Inclou **tots els nodes fins a l'arrel** i, per tant, és **únic a tota la xarxa**.

### 6.2 Tipus de noms de domini

| Tipus | Descripció | Exemple |
|---|---|---|
| **Domini arrel** | No té etiqueta. Es representa amb un **punt**. | `.` |
| **Domini absolut (FQDN)** | Inclou tots els nodes fins a l'arrel. **Acaba en punt.** | `admin.jaumehuguet.xtec.cat.` |
| **Domini relatiu** | No arriba fins a l'arrel. Es completa amb el domini on som. | `admin` (dins de `jaumehuguet.xtec.cat`) |

### 6.3 Com es llegeix un FQDN

Agafem `admin.jaumehuguet.xtec.cat.` i el llegim **de dreta a esquerra**:

```text
   admin  .  jaumehuguet  .  xtec  .  cat  .
     │            │            │       │    │
     │            │            │       │    └── arrel
     │            │            │       └─────── TLD (1r nivell)
     │            │            └─────────────── 2n nivell
     │            └──────────────────────────── 3r nivell
     └───────────────────────────────────────── 4t nivell (equip)
```

> **El punt final és important!** Quan configurem un servidor DNS, un nom **amb punt final** és absolut; un nom **sense punt final** és relatiu i el servidor hi afegirà el domini de la zona. És una de les errades més típiques (ho veurem a la Part 3).

---

## 7. Els dominis de primer nivell (TLD)

A Internet, l'arrel es divideix en **TLD** (*Top Level Domains*). Primer es van crear per **funció** i més tard s'hi van afegir els **geogràfics**.

| TLD funcionals (originals) | TLD nous | TLD geogràfics (ISO 3166) |
|---|---|---|
| `.com` → empreses | `.cat` → llengua i cultura catalanes | `.es` → Espanya |
| `.edu` → educació (EUA) | `.info` → informació | `.fr` → França |
| `.gov` → govern (EUA) | `.biz` → negocis | `.uk` → Regne Unit |
| `.mil` → militar (EUA) | `.name` → persones | `.ad` → Andorra |
| `.org` → organitzacions | `.pro`, `.aero`, `.coop`, `.museum` | `.de` → Alemanya |
| `.net` → xarxes | | `.it` → Itàlia |
| `.int` → organismes internacionals | | |

> **Curiositat:** `.cat` (2005) va ser el primer domini de primer nivell aprovat per a una **comunitat lingüística i cultural** i no per a un estat.

---

## 8. Els servidors arrel

Els **servidors arrel** (*root servers*) són a dalt de tot de l'arbre. **Coneixen on són els servidors de tots els TLD** (`.com`, `.es`, `.cat`...). Sense ells, el DNS no podria funcionar.

| Dada | Explicació |
|---|---|
| **13 servidors arrel** | Hi ha 13 "identitats" de servidor arrel, anomenades de la **A** a la **M** (`a.root-servers.net` ... `m.root-servers.net`). |
| **Anycast** | Cadascuna d'aquestes 13 identitats té **moltes còpies (rèpliques)** repartides per tot el món que comparteixen la mateixa IP. Quan fas una consulta, et respon la rèplica **més propera**. |

> **Nota del professor:** tot i que es parla de "13 servidors", en realitat avui hi ha **més de 1.500 rèpliques físiques** gràcies a l'anycast. Per això el sistema aguanta l'enorme volum de consultes de tot el planeta.

---

## 9. Avantatges del DNS

| Avantatge | Descripció |
|---|---|
| **Sense col·lisions** | L'estructura jeràrquica garanteix que **no hi hagi noms duplicats**. |
| **Escalabilitat** | El model distribuït permet créixer des d'una xarxa local fins a tot Internet. |
| **Independència** | Els noms són independents de les IP: es poden canviar IP sense que l'usuari ho noti. |
| **Versatilitat** | Identifica servidors web, de correu, d'impressió i molts altres serveis. |

**El problema d'escalabilitat i la seva solució:**

- **Model centralitzat (inviable):** un sol equip amb tots els noms, que s'ha de repartir a tothom cada vegada que hi ha un canvi.
- **Model distribuït (DNS):** cada servidor és responsable **només d'una part de l'arbre** (una **zona**). Ho veurem a la Part 2.

---

## 10. Comandes útils de Linux

### 10.1 Resolució de noms

**`dig`** – eina avançada, mostra molta informació de la resposta:

```bash
dig google.com          # Consulta l'adreça IP (registre A)
dig google.com MX       # Consulta els servidors de correu
dig -x 8.8.8.8          # Resolució inversa (IP → nom)
dig @1.1.1.1 google.com # Pregunta a un servidor DNS concret
```

**`nslookup`** – consulta senzilla (també existeix a Windows):

```bash
nslookup google.com           # Usa el DNS configurat a l'equip
nslookup google.com 8.8.8.8   # Pregunta al DNS 8.8.8.8
```

**`host`** – l'eina més simple:

```bash
host google.com          # Resol el nom
host -t MX google.com    # Consulta els registres MX
host 8.8.8.8             # Resolució inversa
```

### 10.2 Verificació i diagnòstic

```bash
# Comprova la connectivitat (i de pas resol el nom)
ping www.google.com

# Mostra el fitxer de noms local (sistema pla)
cat /etc/hosts
sudo nano /etc/hosts          # Editar-lo

# Mostra quins servidors DNS fa servir l'equip
cat /etc/resolv.conf
#   nameserver 8.8.8.8
#   nameserver 8.8.4.4
```

> **Ordre de consulta:** normalment l'equip mira **primer `/etc/hosts`** i, si no hi troba el nom, pregunta al **servidor DNS** indicat a `/etc/resolv.conf`. Aquest ordre es defineix al fitxer `/etc/nsswitch.conf` (línia `hosts: files dns`).

---

## 11. Glossari

| Terme | Definició |
|---|---|
| **DNS** | *Domain Name System*. Sistema jeràrquic i distribuït que tradueix noms de domini a IP i viceversa. |
| **FQDN** | *Fully Qualified Domain Name*. Nom de domini complet i únic que inclou tots els nodes fins a l'arrel (acaba en punt). |
| **TLD** | *Top Level Domain*. Domini de primer nivell (`.com`, `.cat`, `.es`...). |
| **IDNA** | *Internationalized Domain Names*. Mecanisme per fer servir altres alfabets i accents als dominis. |
| **Anycast** | Tècnica que permet tenir moltes rèpliques d'un servidor amb la mateixa IP; respon la més propera. |
| **/etc/hosts** | Fitxer de noms pla de GNU/Linux, hereu del `hosts.txt` d'ARPANET. |
| **Espai de noms** | Conjunt de tots els dominis de l'arbre DNS. |

---

## 12. Resum

| Concepte | Idea clau |
|---|---|
| **Què és** | Servei que tradueix noms ↔ IP. Les persones recordem noms; les màquines, números. |
| **Origen** | D'ARPANET i el fitxer `hosts.txt` (pla i centralitzat) al DNS (jeràrquic i distribuït). |
| **Estructura** | Arbre invertit: arrel (`.`) → TLD → 2n nivell → subdominis/equips. |
| **FQDN** | Nom complet i únic, es llegeix de dreta a esquerra i acaba en punt. |
| **Infraestructura** | 13 identitats de servidors arrel amb moltes rèpliques anycast arreu del món. |

---

## 13. Preguntes de repàs

1. Què significa DNS i per a què serveix? Posa'n un exemple de la vida quotidiana.
2. Explica dos avantatges de la independència entre el nom de domini i l'adreça IP.
3. Què era el fitxer `hosts.txt`? Quin és el seu equivalent actual a Linux? I a Windows?
4. Escriu una línia de `/etc/hosts` perquè l'equip `192.168.1.50` es pugui anomenar `impressora` i `hp`.
5. Per què un sistema de noms pla no es pot fer servir a Internet? Dona'n almenys tres motius.
6. Dibuixa l'arbre DNS del nom `www.jaumehuguet.xtec.cat.` indicant el nivell de cada node.
7. Quina diferència hi ha entre un domini absolut (FQDN) i un domini relatiu?
8. Quins caràcters es poden fer servir en un nom de domini? Quina longitud màxima té cada etiqueta?
9. Classifica aquests TLD en funcionals, nous o geogràfics: `.org`, `.cat`, `.fr`, `.edu`, `.info`, `.ad`.
10. Què fan els servidors arrel? Què vol dir que fan servir *anycast*?
11. En quin ordre es llegeix una ruta en el sistema de fitxers i en el DNS?
12. Quina comanda faries servir per saber quins servidors de correu té el domini `gmail.com`?
