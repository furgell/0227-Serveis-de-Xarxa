# Serveis de Xarxa · RA2 · DNS (Part 3)

## Fitxers de zona i tipus de registres DNS

**Cicle:** 2n SMX (Sistemes Microinformàtics i Xarxes)
**Mòdul:** Serveis de Xarxa
**Resultat d'aprenentatge 2:** Instal·lació i configuració de serveis de resolució de noms (DNS)

---

## Índex

1. [La base de dades del DNS](#1-la-base-de-dades-del-dns)
2. [Els fitxers de zona](#2-els-fitxers-de-zona)
3. [Què és un registre de recurs (RR)?](#3-què-és-un-registre-de-recurs-rr)
4. [Registre SOA (Start of Authority)](#4-registre-soa-start-of-authority)
5. [Registre NS (Name Server)](#5-registre-ns-name-server)
6. [Registre A (Address)](#6-registre-a-address)
7. [Registre CNAME (Canonical Name)](#7-registre-cname-canonical-name)
8. [Registre PTR (Pointer)](#8-registre-ptr-pointer)
9. [Registre MX (Mail eXchanger)](#9-registre-mx-mail-exchanger)
10. [Altres registres](#10-altres-registres)
11. [Abreviacions en els fitxers de zona](#11-abreviacions-en-els-fitxers-de-zona)
12. [Exemple complet de fitxers de zona](#12-exemple-complet-de-fitxers-de-zona)
13. [Resolució directa vs. inversa](#13-resolució-directa-vs-inversa)
14. [Errades típiques](#14-errades-típiques)
15. [Resum](#15-resum)
16. [Preguntes de repàs](#16-preguntes-de-repàs)

---

## 1. La base de dades del DNS

El DNS és una **base de dades jeràrquica i distribuïda**: cada servidor de noms només guarda la informació de la **zona** de la qual és **autoritatiu** (Part 2).

Cada zona conté la informació dels equips (*hosts*) que en formen part. Aquesta informació es guarda en **registres de recurs** (*Resource Records*, **RR**), que serveixen per a:

| Funció | Exemple |
|---|---|
| **Resolució directa** (*forward mapping*): nom → IP | `mahatma.inf.ioc.cat` → `192.168.0.2` |
| **Resolució inversa** (*reverse mapping*): IP → nom | `192.168.0.2` → `mahatma.inf.ioc.cat` |
| **Identificar servidors** | Quins són els servidors DNS i de correu de la zona |

> **Idea clau:** el conjunt de **tots els RR de totes les zones** del món forma la gran base de dades distribuïda del DNS.

---

## 2. Els fitxers de zona

La configuració d'una zona es guarda en **fitxers de zona**: fitxers de text amb un format definit per l'estàndard DNS.

### 2.1 Fitxers mínims d'una zona

Qualsevol servidor DNS té, com a mínim, aquests fitxers:

| # | Fitxer | Què conté |
|---|---|---|
| **1** | **Resolució directa** | Les associacions **nom → IP** de tots els equips de la zona. |
| **2** | **Resolució inversa** | Un fitxer **per a cada subxarxa** amb les associacions **IP → nom**. |
| **3** | **Loopback** | La resolució inversa de l'adreça de *loopback* (`127.0.0.1` → `localhost`). |
| **4** | **Nodes arrel** (*root hints*) | La llista dels **servidors arrel** d'Internet i les seves IP, perquè el servidor sàpiga per on començar a buscar. |

> Els fitxers de **loopback** i de **nodes arrel** són sempre iguals i normalment **ja venen fets** amb el programa (o es copien d'una altra instal·lació). Els que hem d'escriure nosaltres són els de **resolució directa i inversa**. Un cop fets, cal **indicar al servidor de noms que els faci servir** (a BIND, al fitxer `named.conf.local`).

---

## 3. Què és un registre de recurs (RR)?

### 3.1 Sintaxi general

Cada línia d'un fitxer de zona que comença a la **primera columna** defineix un registre:

```text
 domini      classe   [ttl]   tipus    rdata
```

| Camp | Significat |
|---|---|
| **domini** | El nom que s'està definint (la "pregunta"). |
| **classe** | Actualment sempre **`IN`** (*Internet*). |
| **ttl** | *Opcional.* Temps (en segons) que el registre pot quedar a la **cache** d'altres servidors. Normalment es posa un TTL general per a tot el fitxer amb `$TTL`. |
| **tipus** | Tipus de registre: `SOA`, `NS`, `A`, `CNAME`, `PTR`, `MX`... |
| **rdata** | El **valor** associat (la "resposta"): una IP, un nom, etc. |

**Exemple:**

```text
 mahatma.inf.ioc.cat.   IN   3600   A   192.168.0.2
 └──── domini ──────┘ classe  ttl  tipus └─ rdata ─┘
```

> **Truc per recordar-ho:** a l'**esquerra** hi ha **el que preguntes** i a la **dreta** **el que et responen**.

### 3.2 Els registres més habituals

| Tipus | Nom complet | Per a què serveix |
|---|---|---|
| **SOA** | *Start of Authority* | Inici de la zona: diu que aquest servidor n'és l'autoritat. |
| **NS** | *Name Server* | Indica quins són els servidors DNS de la zona. |
| **A** | *Address* | Nom → adreça IPv4. |
| **CNAME** | *Canonical Name* | Àlies → nom real. |
| **PTR** | *Pointer* | IP → nom (resolució inversa). |
| **MX** | *Mail eXchanger* | Servidor de correu del domini. |

---

## 4. Registre SOA (Start of Authority)

### 4.1 Què és

El registre **SOA** indica que aquest fitxer de zona és **la millor font de dades** de la zona, és a dir, que el servidor hi té **autoritat**.

- Acostuma a ser el **primer registre** del fitxer.
- **Només n'hi pot haver un per zona.**

### 4.2 Sintaxi

```text
nomDomini.  IN  SOA  nsPrimari.  admin.nsPrimari.  ( serial refresh retry expire minimum )
```

**Exemple:**

```text
inf.ioc.cat.  IN  SOA  ns1.inf.ioc.cat.  admin.ns1.inf.ioc.cat.  ( 2026110501 3h 1h 1w 1h )
```

> **Nota del professor:** a la presentació, l'exemple és `( 1h 3h 1h 1w 1h )`. Fixa't que el **primer valor és el serial**, que és un **número de versió** i no un temps. Per això aquí hi posem `2026110501`.

| Camp | Exemple | Significat |
|---|---|---|
| **nomDomini.** | `inf.ioc.cat.` | El domini del qual el servidor és autoritatiu. **El punt final és obligatori.** |
| **nsPrimari.** | `ns1.inf.ioc.cat.` | El servidor de noms **primari** de la zona (amb punt final). |
| **admin.nsPrimari.** | `admin.ns1.inf.ioc.cat.` | El **correu de l'administrador**, escrit amb un punt en lloc de l'`@`. |

> **Compte amb el correu!** El **primer punt** equival a l'**`@`**. Així, `admin.ns1.inf.ioc.cat.` vol dir **`admin@ns1.inf.ioc.cat`**. No es pot escriure l'`@` perquè en els fitxers de zona té un altre significat (ho veurem a l'apartat 11).

### 4.3 Les opcions per als secundaris

Els valors entre parèntesis regulen la comunicació entre el **servidor primari (master)** i els **secundaris (slaves)**:

| Paràmetre | Exemple | Significat |
|---|---|---|
| **Serial** | `2026110501` | **Número de versió** de les dades. Cada cop que es modifica la zona **cal augmentar-lo**; així el secundari sap que hi ha canvis i els copia. |
| **Refresh** | `3h` | Cada quant temps el **secundari comprova** si hi ha canvis al primari. |
| **Retry** | `1h` | Si el refresc ha fallat, quant temps **espera abans de tornar-ho a intentar**. Normalment és més petit que el *refresh*. |
| **Expire** | `1w` | Si el secundari no aconsegueix contactar amb el primari durant aquest temps, considera que les seves dades **ja no són vàlides** i deixa de respondre amb autoritat. |
| **Minimum** | `1h` | TTL per defecte de les respostes **negatives** (quan un nom no existeix). |

**Unitats de temps:** `s` segons, `m` minuts, `h` hores, `d` dies, `w` setmanes. Si no hi poses unitat, són **segons** (`3600` = `1h`).

> **Consell: format del serial.** Es fa servir molt el format **`AAAAMMDDnn`** (any, mes, dia i número de canvi del dia). Exemple: `2026110501` = 5 de novembre de 2026, primer canvi. **Si oblides augmentar el serial, el secundari no s'actualitzarà!**

---

## 5. Registre NS (Name Server)

### 5.1 Què és

El registre **NS** indica quin és un **servidor de noms autoritatiu** de la zona.

- Hi haurà **una entrada NS per a cada servidor** autoritatiu.
- L'estàndard en recomana **almenys dos**: un **primari (master)** i un **secundari (slave)**.

### 5.2 Sintaxi

```text
nomDomini.   IN   NS   nameServer.
```

**Exemple:**

```text
inf.ioc.cat.   IN   NS   ns1.inf.ioc.cat.
inf.ioc.cat.   IN   NS   ns2.inf.ioc.cat.
```

| Camp | Significat |
|---|---|
| **nomDomini.** | El domini que s'està definint. |
| **NS** | Tipus de registre: servidor de noms. |
| **nameServer.** | Nom del servidor de noms (**punt final obligatori**). |

> Els registres NS també serveixen per fer **delegacions**: si a la zona `ioc.cat` hi posem `inf.ioc.cat. IN NS ns1.inf.ioc.cat.`, estem dient que la zona `inf.ioc.cat` la gestiona un altre servidor.

---

## 6. Registre A (Address)

### 6.1 Què és

El registre **A** associa un **nom d'equip** a una **adreça IPv4**. És el registre de la **resolució directa**. Cal **una entrada A per a cada equip** de la xarxa.

### 6.2 Sintaxi

```text
nomHost.   IN   A   IP
```

**Exemple:**

```text
mahatma.inf.ioc.cat.   IN   A   192.168.0.2
```

### 6.3 Equips amb diverses IP (*multihomed*)

Un equip pot tenir **més d'una IP** (per exemple, un servidor amb dues targetes de xarxa). En aquest cas es posa **un registre A per a cada IP**, amb el **mateix nom** a l'esquerra:

```text
superserver.dom.com.   IN   A   10.0.0.1
superserver.dom.com.   IN   A   10.0.0.2
```

| Cas | Registres A |
|---|---|
| **Host simple** | Un registre A (una IP). |
| **Host multihomed** | Un registre A per a cada IP, amb el mateix nom. |

> **Important:** els noms definits amb registres A són els **noms canònics** (els noms "reals" dels equips). Si un equip ha de tenir altres noms (àlies), es defineixen amb registres **CNAME**.

---

## 7. Registre CNAME (Canonical Name)

### 7.1 Què és

El registre **CNAME** defineix un **àlies**: un **nom alternatiu** per a un equip que ja té un nom canònic.

És molt útil per donar noms de **servei** als equips. Per exemple, el servidor `mahatma` pot fer de servidor web i FTP alhora:

```text
www.inf.ioc.cat  ──┐
                   ├──▶  mahatma.inf.ioc.cat  ──▶  192.168.0.2
ftp.inf.ioc.cat  ──┘
   (àlies)               (nom canònic, registre A)
```

La resolució segueix una **cadena**:

```text
  àlies  →  CNAME  →  nom canònic  →  A  →  adreça IP
```

### 7.2 Sintaxi

```text
nomAlies.   IN   CNAME   nomCanonic.
```

**Exemples:**

```text
ftp.inf.ioc.cat.   IN   CNAME   mahatma.inf.ioc.cat.
www.inf.ioc.cat.   IN   CNAME   mahatma.inf.ioc.cat.
```

| Camp | Significat |
|---|---|
| **nomAlies.** | El nom de l'àlies que estem creant. |
| **CNAME** | Tipus de registre: àlies. |
| **nomCanonic.** | El **nom canònic** (el que té el registre A) al qual apunta l'àlies. |

### 7.3 Normes importants del CNAME

1. **Un CNAME sempre apunta a un nom, mai a una IP.** Si vols associar un nom directament a una IP, fes servir un registre **A**.
2. **Un àlies mai pot aparèixer a la part dreta d'un altre registre** (NS, MX, PTR...). A la dreta hi ha d'anar sempre el **nom canònic**.
3. Un nom que té un CNAME **no pot tenir cap altre registre** (ni A, ni MX...).

> **Nota del professor:** a la presentació apareix l'exemple `tftp.inf.ioc.cat IN CNAME 192.168.0.2`. **Aquest exemple és incorrecte** per dos motius: un CNAME no pot apuntar a una IP i, a més, al nom li falta el punt final. La forma correcta seria:
> ```text
> tftp.inf.ioc.cat.   IN   CNAME   mahatma.inf.ioc.cat.
> ```
> o bé, si realment volem associar el nom a una IP:
> ```text
> tftp.inf.ioc.cat.   IN   A       192.168.0.2
> ```

---

## 8. Registre PTR (Pointer)

### 8.1 Què és

El registre **PTR** associa una **adreça IP** al **nom de l'equip**. És el registre de la **resolució inversa** (el contrari del registre A).

Cal **una entrada PTR per a cada interfície de xarxa** (cada IP) de la zona.

### 8.2 Sintaxi

```text
ipInversa.in-addr.arpa.   IN   PTR   nomCanonic.
```

Recorda (Part 2): per escriure la IP en format `in-addr.arpa` s'**inverteixen els octets**.

**Exemple:** l'equip `mahatma.inf.ioc.cat` té la IP `192.168.0.2`:

```text
   IP:          192.168.0.2
   Invertida:   2.0.168.192.in-addr.arpa.
```

```text
2.0.168.192.in-addr.arpa.   IN   PTR   mahatma.inf.ioc.cat.
```

| Camp | Significat |
|---|---|
| **ipInversa.in-addr.arpa.** | La IP amb els octets invertits i el sufix `in-addr.arpa.` |
| **PTR** | Tipus de registre: resolució inversa. |
| **nomCanonic.** | El nom **canònic** complet (FQDN) de l'equip. Mai un àlies! |

> **Nota del professor:** a la presentació, l'exemple de PTR fa servir `2.20.168.192.in-addr.arpa.`, que correspondria a la IP `192.168.20.2`, mentre que el registre A de `mahatma` és `192.168.0.2`. Perquè siguin coherents, el PTR ha de ser `2.0.168.192.in-addr.arpa.`, com a l'exemple d'aquí dalt.

---

## 9. Registre MX (Mail eXchanger)

### 9.1 Què és

El registre **MX** indica quin és el **servidor de correu electrònic** d'un domini. Quan algú envia un correu a `alumne@inf.ioc.cat`, el seu servidor de correu consulta el registre MX de `inf.ioc.cat` per saber **a quin servidor l'ha de lliurar**.

- Es pot posar **una entrada MX per a cada servidor de correu**.
- **No és obligatori** tenir-ne cap (si el domini no rep correu).

### 9.2 Sintaxi

```text
nomDomini.   IN   MX   prioritat   servidorCorreu.
```

**Exemple:**

```text
inf.ioc.cat.   IN   MX   10   correu1.inf.ioc.cat.
inf.ioc.cat.   IN   MX   20   correu2.inf.ioc.cat.
```

### 9.3 La prioritat

El número de **prioritat** indica en quin ordre s'han de fer servir els servidors:

- **Com més petit és el número, més prioritat** té el servidor.
- En l'exemple, primer s'intenta lliurar el correu a `correu1` (10). Si no respon, s'envia a `correu2` (20), que fa de **servidor de reserva**.

> **Recorda:** el servidor de la dreta d'un MX ha de ser un **nom canònic** (amb registre A), **mai un àlies (CNAME)**.

---

## 10. Altres registres

A més dels principals, l'estàndard DNS defineix altres tipus de registre:

| Tipus | Significat | Ús actual |
|---|---|---|
| **AAAA** | Adreça **IPv6** d'un equip (com el registre A, però per a IPv6). | ✅ Molt utilitzat. |
| **TXT** | **Text** descriptiu associat al domini. | ✅ Molt utilitzat (verificació de dominis, antispam SPF/DKIM). |
| **SRV** | Localitza **serveis** concrets (per exemple, el controlador de domini d'Active Directory). | ✅ Utilitzat. |
| **HINFO** | *Host Information*: tipus de maquinari i sistema operatiu. | ⚠️ Poc usat (dona informació a possibles atacants). |
| **WKS** | *Well Known Services*: llista de serveis de l'equip. | ❌ Obsolet. |
| **MB / MG / MR** | Bústia de correu, grup de correu i nou nom de bústia. | ❌ Obsolets. |
| **NULL** | Registre buit. | ❌ Experimental. |

**Exemples:**

```text
mahatma.inf.ioc.cat.   IN   AAAA   2001:db8::2
inf.ioc.cat.           IN   TXT    "Servidor DNS de l'aula d'informàtica"
```

---

## 11. Abreviacions en els fitxers de zona

Escriure noms complets a cada línia és pesat i fàcil d'equivocar-se. L'estàndard permet **tres abreviacions**:

| Abreviació | Què vol dir | Exemple |
|---|---|---|
| **`@`** | Representa el **nom de la zona** (el domini d'origen). | `@  IN  NS  ns1.inf.ioc.cat.` equival a `inf.ioc.cat.  IN  NS  ns1.inf.ioc.cat.` |
| **Camp buit** | Si la primera columna comença amb **espais o tabulador**, s'usa el **mateix nom que el registre anterior**. | Vegeu l'exemple de sota. |
| **Noms relatius** | A un nom que **no acaba en punt** se li **afegeix automàticament** el nom de la zona. | `mahatma` → `mahatma.inf.ioc.cat.` |

**Exemple amb les tres abreviacions** (zona `inf.ioc.cat`):

```text
$ORIGIN inf.ioc.cat.
@          IN   NS      ns1            ; @ = inf.ioc.cat.  /  ns1 = ns1.inf.ioc.cat.
           IN   NS      ns2            ; camp buit = inf.ioc.cat. (igual que l'anterior)
           IN   MX  10  correu         ; camp buit = inf.ioc.cat.
mahatma    IN   A       192.168.0.2    ; mahatma = mahatma.inf.ioc.cat.
ftp        IN   CNAME   mahatma        ; ftp.inf.ioc.cat. → mahatma.inf.ioc.cat.
```

> **ALERTA – la trampa del punt final:**
> ```text
> www   IN   CNAME   mahatma.inf.ioc.cat     ; ❌ falta el punt final!
> ```
> Com que `mahatma.inf.ioc.cat` no acaba en punt, el servidor el considera **relatiu** i hi afegeix el nom de la zona:
> `mahatma.inf.ioc.cat.inf.ioc.cat.` 😱
>
> **Regla d'or:** o escrius el nom **curt i sense punt** (`mahatma`), o l'escrius **complet i amb punt** (`mahatma.inf.ioc.cat.`). Mai complet i sense punt.

---

## 12. Exemple complet de fitxers de zona

Volem configurar el DNS d'una xarxa d'aula:

| Equip | Funció | IP |
|---|---|---|
| `ns1` | Servidor DNS primari | `192.168.0.1` |
| `ns2` | Servidor DNS secundari | `192.168.0.3` |
| `mahatma` | Servidor web i FTP | `192.168.0.2` |
| `correu` | Servidor de correu | `192.168.0.10` |
| `pc01` | Equip d'alumne | `192.168.0.101` |

### 12.1 Fitxer de resolució directa (`db.inf.ioc.cat`)

```text
$TTL 1h                                   ; TTL per defecte de tot el fitxer
$ORIGIN inf.ioc.cat.

@   IN  SOA  ns1.inf.ioc.cat.  admin.inf.ioc.cat. (
             2026110501   ; Serial  (AAAAMMDDnn)
             3h           ; Refresh
             1h           ; Retry
             1w           ; Expire
             1h )         ; Minimum

; ---- Servidors de noms ----
@         IN  NS      ns1
@         IN  NS      ns2

; ---- Servidor de correu ----
@         IN  MX  10  correu

; ---- Equips (noms canònics) ----
ns1       IN  A       192.168.0.1
mahatma   IN  A       192.168.0.2
ns2       IN  A       192.168.0.3
correu    IN  A       192.168.0.10
pc01      IN  A       192.168.0.101

; ---- Àlies ----
www       IN  CNAME   mahatma
ftp       IN  CNAME   mahatma
```

### 12.2 Fitxer de resolució inversa (`db.192.168.0`)

```text
$TTL 1h
$ORIGIN 0.168.192.in-addr.arpa.

@   IN  SOA  ns1.inf.ioc.cat.  admin.inf.ioc.cat. (
             2026110501 3h 1h 1w 1h )

@     IN  NS   ns1.inf.ioc.cat.
@     IN  NS   ns2.inf.ioc.cat.

1     IN  PTR  ns1.inf.ioc.cat.
2     IN  PTR  mahatma.inf.ioc.cat.
3     IN  PTR  ns2.inf.ioc.cat.
10    IN  PTR  correu.inf.ioc.cat.
101   IN  PTR  pc01.inf.ioc.cat.
```

> **Fixa't:**
> - Al fitxer **invers**, la part esquerra és només l'**últim octet** (`2`), que és relatiu i es converteix en `2.0.168.192.in-addr.arpa.`
> - La part dreta dels PTR s'ha d'escriure **completa i amb punt**, perquè la zona d'aquest fitxer és `0.168.192.in-addr.arpa.` i no `inf.ioc.cat.`
> - Els àlies (`www`, `ftp`) **no** tenen PTR: només els noms canònics.

### 12.3 Comprovar-ho

```bash
# Comprovar la sintaxi dels fitxers (BIND)
named-checkzone inf.ioc.cat /etc/bind/db.inf.ioc.cat
named-checkzone 0.168.192.in-addr.arpa /etc/bind/db.192.168.0

# Fer consultes al nostre servidor
dig @192.168.0.1 mahatma.inf.ioc.cat        # registre A
dig @192.168.0.1 www.inf.ioc.cat            # CNAME → A
dig @192.168.0.1 inf.ioc.cat MX             # servidor de correu
dig @192.168.0.1 inf.ioc.cat NS             # servidors de noms
dig @192.168.0.1 -x 192.168.0.2             # resolució inversa (PTR)
```

---

## 13. Resolució directa vs. inversa

| | **Resolució directa** (*forward mapping*) | **Resolució inversa** (*reverse mapping*) |
|---|---|---|
| **Pregunta** | Quina IP té aquest nom? | Quin nom té aquesta IP? |
| **Transforma** | Nom → IP | IP → nom |
| **Registres** | **A** (i **CNAME** per als àlies) | **PTR** |
| **Exemple** | `mahatma.inf.ioc.cat` → `192.168.0.2` | `2.0.168.192.in-addr.arpa` → `mahatma.inf.ioc.cat` |

### Flux de resolució

```text
  ① CONSULTA          ② RECERCA              ③ RESOLUCIÓ                ④ RESPOSTA
  El client envia  →  El servidor busca  →  Retorna el registre A,  →  El client rep la
  el nom de domini    al fitxer de zona     o segueix un CNAME / PTR    IP i es connecta
```

---

## 14. Errades típiques

| Errada | Conseqüència | Com evitar-la |
|---|---|---|
| Oblidar el **punt final** en un nom complet | El servidor hi afegeix el nom de la zona (`nom.zona.zona.`). | Nom curt sense punt, o complet amb punt. |
| No **augmentar el serial** del SOA | El secundari no copia els canvis. | Augmentar-lo sempre que es modifica la zona. |
| Posar `@` al correu de l'administrador | Error de sintaxi o correu incorrecte. | Escriure `admin.domini.` en lloc d'`admin@domini`. |
| **CNAME** que apunta a una IP | Registre invàlid. | El CNAME sempre apunta a un nom. Per a IP, registre A. |
| **MX** o **NS** que apunten a un àlies | Funcionament incorrecte. | Apuntar sempre a noms canònics. |
| Escriure la IP **sense invertir** al PTR | La resolució inversa no funciona. | `192.168.0.2` → `2.0.168.192.in-addr.arpa.` |
| Afegir un equip al fitxer directe però **no a l'invers** | El nom funciona però la IP no resol. | Mantenir els dos fitxers sincronitzats. |

---

## 15. Resum

| Concepte | Idea clau |
|---|---|
| **Base de dades distribuïda** | Cada zona guarda els seus registres en fitxers de zona estandarditzats. |
| **Fitxers mínims** | Directa, inversa (un per subxarxa), loopback i nodes arrel. |
| **Sintaxi d'un RR** | `domini  classe  [ttl]  tipus  rdata` |
| **SOA** | Inici de la zona autoritativa. Un per zona. Conté el serial i els temps per als secundaris. |
| **NS** | Servidors de noms de la zona. Mínim 2 recomanats. |
| **A** | Nom → IPv4 (resolució directa). Defineix noms canònics. |
| **CNAME** | Àlies → nom canònic. Mai a una IP. |
| **PTR** | IP invertida + `in-addr.arpa` → nom (resolució inversa). |
| **MX** | Servidor de correu amb prioritat (número més petit = més prioritat). Opcional. |
| **Abreviacions** | `@` = nom de la zona; camp buit = nom anterior; sense punt = s'hi afegeix la zona. |

---

## 16. Preguntes de repàs

1. Què és un registre de recurs (RR)? Escriu la seva sintaxi general i explica cada camp.
2. Quins quatre fitxers de zona hi ha, com a mínim, en un servidor DNS? Quins hem d'escriure nosaltres?
3. Quants registres SOA pot haver-hi en una zona? On se sol posar?
4. Com s'escriu el correu `jefe@empresa.cat` dins d'un registre SOA? Per què?
5. Explica què signifiquen els paràmetres *serial*, *refresh*, *retry* i *expire* del SOA.
6. Què passa si modifiques un fitxer de zona i oblides augmentar el serial?
7. Escriu els registres NS de la zona `aula.local` amb els servidors `dns1` i `dns2`.
8. Escriu els registres A d'un servidor `firewall.aula.local` que té les IP `10.0.0.1` i `192.168.1.1`.
9. Crea un àlies `intranet.aula.local` que apunti al servidor `srv01.aula.local`. Per què seria incorrecte fer-lo apuntar a `10.0.0.5`?
10. Escriu el registre PTR de l'equip `pc15.aula.local`, que té la IP `192.168.1.115`.
11. Un domini té dos registres MX amb prioritats 5 i 50. A quin servidor s'enviarà primer el correu?
12. Per a què serveix el registre AAAA? I el TXT?
13. Dins la zona `aula.local.`, a quin nom complet correspon cadascun d'aquests: `@`, `pc01`, `pc01.aula.local`, `pc01.aula.local.`?
14. Troba les errades d'aquest fragment de la zona `aula.local.`:
    ```text
    @      IN  SOA  dns1.aula.local  admin@aula.local. ( 1 3h 1h 1w 1h )
    @      IN  NS   web
    web    IN  CNAME 10.0.0.5
    @      IN  MX   10  web
    ```
