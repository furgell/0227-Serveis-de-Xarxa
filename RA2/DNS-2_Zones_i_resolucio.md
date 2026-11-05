# Serveis de Xarxa · RA2 · DNS (Part 2)

## Zones, servidors i mecanisme de resolució

**Cicle:** 2n SMX (Sistemes Microinformàtics i Xarxes)
**Mòdul:** Serveis de Xarxa
**Resultat d'aprenentatge 2:** Instal·lació i configuració de serveis de resolució de noms (DNS)

---

## Índex

1. [Arquitectura client-servidor](#1-arquitectura-client-servidor)
2. [Què és una zona DNS?](#2-què-és-una-zona-dns)
3. [La delegació de zones](#3-la-delegació-de-zones)
4. [Servidors primari i secundari](#4-servidors-primari-i-secundari)
5. [El resolver: la part client del DNS](#5-el-resolver-la-part-client-del-dns)
6. [Tipus de respostes DNS](#6-tipus-de-respostes-dns)
7. [Resolució recursiva i iterativa](#7-resolució-recursiva-i-iterativa)
8. [La cache DNS](#8-la-cache-dns)
9. [La resolució inversa (in-addr.arpa)](#9-la-resolució-inversa-in-addrarpa)
10. [El protocol DNS](#10-el-protocol-dns)
11. [Estructura d'un missatge DNS](#11-estructura-dun-missatge-dns)
12. [DDNS: DNS dinàmic](#12-ddns-dns-dinàmic)
13. [DNSSEC: seguretat al DNS](#13-dnssec-seguretat-al-dns)
14. [Resum](#14-resum)
15. [Preguntes de repàs](#15-preguntes-de-repàs)

---

## 1. Arquitectura client-servidor

El DNS funciona amb el model **client-servidor**:

- Els **clients** fan preguntes del tipus: *«Quina IP té `www.xtec.cat`?»*
- Els **servidors DNS** (també anomenats **servidors de noms** o *name servers*) responen.

Cap servidor sap tots els noms del món. Cada servidor **emmagatzema i gestiona** la informació d'una **part** de l'espai de noms, anomenada **zona**, i sobre aquesta part té **autoritat**.

```text
   CLIENT                                 SERVIDOR DNS
     │ ── "Quina IP té www.xtec.cat?" ──▶ │
     │ ◀──────── "83.247.151.214" ─────── │
```

---

## 2. Què és una zona DNS?

### 2.1 Definició

Una **zona** és la **part de l'espai de noms** que gestiona un servidor DNS (o un grup de servidors). Els servidors que gestionen una zona tenen la **informació completa** sobre ella i es diu que hi tenen **autoritat** (són **autoritatius** o **autoritaris**).

### 2.2 Domini ≠ Zona

Aquesta és una de les idees que més costa entendre:

| | **Domini** | **Zona** |
|---|---|---|
| **Què és** | Un node i **tot** l'arbre que penja d'ell. | La part de l'arbre que **administra un servidor concret**. |
| **Mida** | Inclou tots els subdominis. | Pot excloure els subdominis que s'han **delegat** a altres. |

Poden **coincidir** (si no hi ha cap delegació), però no sempre és així.

> **Analogia:** pensa en una **escola**. El *domini* és tota l'escola. La *direcció* gestiona l'escola, però ha "delegat" el **departament d'informàtica** al seu cap de departament. La *zona* de la direcció és tota l'escola **menys** el departament d'informàtica, que és una zona a part.

### 2.3 Exemple: un espai de noms amb quatre zones

```text
                                  A
         ┌────────────────────────┼──────────────┬──────┐
         B                        C              D      E
       ┌─┴─┐                ┌─────┼─────┐        │      │
       F   G                H     I     J        K      L
                                ┌─┴─┐
                                M   N
```

| Zona | Nom de la zona | Nodes que conté |
|---|---|---|
| **Zona 1** | `A` | A, D, E, K, L |
| **Zona 2** | `B.A` | B, F, G |
| **Zona 3** | `C.A` | C, H, J |
| **Zona 4** | `I.C.A` | I, M, N |

- L'espai de noms té **14 dominis** (de A a N) repartits en **4 zones**.
- Cada zona rep el nom del seu **node superior**: `A`, `B.A`, `C.A` i `I.C.A`.
- La zona `A` ha **delegat** `B.A` i `C.A`; i la zona `C.A` ha delegat `I.C.A`.
- Fixa't que el **domini `C.A`** inclou C, H, I, J, M i N, però la **zona `C.A`** només inclou C, H i J, perquè I, M i N s'han delegat a la zona `I.C.A`.
- Igualment, el **domini `A`** és tot l'arbre, però la **zona `A`** només conté A, D, E, K i L.

### 2.4 Conceptes clau

| Concepte | Definició |
|---|---|
| **Domini** | L'arbre complet que penja d'un node. |
| **Zona** | La part de l'arbre administrada per un servidor DNS concret. |
| **Base de dades de zona** | Els **fitxers** que guarden la informació dels equips de la zona (ho veurem a la Part 3). |
| **Delegació** | Passar l'autoritat d'un subdomini a una altra entitat i els seus servidors DNS. |

---

## 3. La delegació de zones

**Delegar** un subdomini vol dir **cedir-ne l'autoritat** a uns altres servidors DNS. A partir d'aquell moment:

- La nova entitat té **tot el control** per administrar la zona delegada com vulgui.
- La **zona pare perd el control administratiu** de la zona delegada.
- La zona pare només guarda **un "punter"** cap als servidors de la zona filla, per saber a qui preguntar.

**Exemple real:**

```text
  .cat  (gestionat per la Fundació puntCAT)
    └── xtec.cat  ← delegat al Departament d'Educació
          └── jaumehuguet.xtec.cat  ← podria delegar-se a l'institut
```

La Fundació puntCAT no sap quins equips hi ha a `xtec.cat`; només sap **quins servidors DNS** en són responsables.

> Gràcies a la delegació, la feina es **reparteix**: ningú no ha de gestionar tot Internet.

---

## 4. Servidors primari i secundari

### 4.1 Per què calen dos servidors (o més)?

L'estàndard DNS diu que cada zona ha de tenir **dos o més servidors autoritatius**. Motius:

| Motiu | Explicació |
|---|---|
| **Redundància** | Si un servidor cau, l'altre continua responent. |
| **Robustesa** | Si el servidor fos únic i fallés, **la xarxa quedaria inoperativa** (ningú no podria resoldre noms). |
| **Rendiment** | Les consultes es reparteixen entre diversos servidors. |
| **Còpia de seguretat** | Hi ha diverses còpies de les dades de la zona. |

### 4.2 Primari vs. secundari

| | **Servidor primari (master)** | **Servidor secundari (slave)** |
|---|---|---|
| **Fitxers de zona** | És **l'únic** que té els fitxers **originals**. | Té una **còpia**. |
| **On es fan els canvis** | Aquí. L'administrador edita els fitxers al primari. | Mai directament. |
| **Com obté les dades** | De l'administrador. | Per **transferència de zona** des del primari. |
| **Respon consultes?** | Sí, amb autoritat. | Sí, també amb autoritat. |

```text
    Administrador
         │ edita
         ▼
   ┌───────────┐   transferència de zona   ┌─────────────┐
   │  PRIMARI  │ ────────────────────────▶ │ SECUNDARI   │
   │ (original)│        (TCP 53)           │   (còpia)   │
   └───────────┘                           └─────────────┘
         ▲                                        ▲
         └──────────── consultes dels clients ────┘
```

> **Important:** el secundari també és **autoritatiu**. Per al client no hi ha diferència entre preguntar al primari o al secundari.

---

## 5. El resolver: la part client del DNS

El **resolver** (*resolutor*) és la **part client** del DNS. És el que fa servir qualsevol programa (navegador, `ping`, client de correu...) quan necessita traduir un nom.

| Característica | Explicació |
|---|---|
| **Funció** | Rep la petició de l'aplicació, **confecciona la consulta** (*query*), l'envia al servidor DNS, rep la resposta i la **retorna a l'aplicació**. |
| **No és un programa** | Normalment no és una aplicació independent, sinó un **conjunt de biblioteques de funcions** del sistema operatiu que les aplicacions fan servir. |
| **Consultes** | El resolver pregunta al servidor DNS configurat a l'equip (a Linux, el de `/etc/resolv.conf`). |

> **Exemple:** quan escrius `www.xtec.cat` al navegador, el navegador **no** sap fer consultes DNS; crida una funció del sistema (el resolver), que és qui pregunta al servidor DNS.

---

## 6. Tipus de respostes DNS

Quan un servidor DNS rep una consulta, pot passar el següent:

| Situació | Tipus de resposta | D'on surt la informació |
|---|---|---|
| El nom és de la **seva zona** | **Autoritativa** | De la **base de dades de la seva zona**. És la resposta "oficial". |
| El nom **no és seu**, però ja l'havia consultat abans | **No autoritativa** | De la **cache** (memòria temporal). |
| El nom no és seu i **no el té a la cache** | **Sense informació** | Ha de preguntar a altres servidors de manera **recursiva** o **iterativa**. |

El contingut de la resposta pot ser:

- ✅ **La dada demanada** (per exemple, la IP).
- ⚠️ **El domini existeix però no té aquella dada** (per exemple, existeix però no té servidor de correu).
- ❌ **El domini no existeix** (error `NXDOMAIN`).

> Quan fas `nslookup` i surt el missatge **"Non-authoritative answer"**, vol dir que la resposta ve de la **cache** d'un servidor que no és el responsable d'aquell domini.

---

## 7. Resolució recursiva i iterativa

### 7.1 Les dues maneres de preguntar

| | **Mode recursiu** | **Mode iteratiu** |
|---|---|---|
| **Què fa el servidor** | Busca la resposta **ell mateix**, preguntant a tants servidors com calgui. | Respon **només amb el que sap** (zona pròpia + cache). **No pregunta a ningú.** |
| **Què rep qui pregunta** | La **resposta definitiva**. | La resposta o bé una **referència**: «no ho sé, però pregunta a aquests servidors, que estan més a prop». |
| **Qui l'utilitza** | Normalment el **client (resolver)** → el seu servidor DNS. | Els **servidors entre ells**. |

> **Analogia:**
> - **Recursiu:** demanes a la teva mare on és una botiga. Ella truca a qui calgui i al final et diu l'adreça exacta.
> - **Iteratiu:** preguntes a un vianant i et diu: «no ho sé, però pregunta a l'oficina de turisme». Vas a l'oficina i et diuen: «pregunta a l'ajuntament»... i així fins que algú t'ho sap dir.

### 7.2 El procés complet pas a pas

Volem resoldre **`www.cult.gva.es`** i el nostre servidor DNS local no en sap res:

```text
                         ┌──────────────────┐
                    ②    │  Servidor ARREL  │
              ┌─────────▶│       (.)        │
              │ ◀─────── └──────────────────┘
              │    ③ "Pregunta als de .es"
              │
              │          ┌──────────────────┐
              │     ④    │  Servidor de .es │
              ├─────────▶│                  │
              │ ◀─────── └──────────────────┘
              │    ⑤ "Pregunta als de gva.es"
┌──────────┐  │
│ Servidor │  │          ┌──────────────────┐
│ DNS LOCAL│──┤     ⑥    │ Servidor gva.es  │
└──────────┘  ├─────────▶│                  │
   ▲    │     │ ◀─────── └──────────────────┘
   │    │     │    ⑦ "Pregunta als de cult.gva.es"
 ① │    │ ⑩   │
   │    ▼     │          ┌──────────────────────┐
┌──────────┐  │     ⑧    │ Servidor cult.gva.es │
│  CLIENT  │  └─────────▶│   (AUTORITATIU)      │
└──────────┘    ◀─────── └──────────────────────┘
                   ⑨ "www.cult.gva.es = 1.2.3.4"
```

1. **①** El client pregunta al seu servidor local en mode **recursiu**: «vull la resposta final».
2. **②–⑨** El servidor local pregunta en mode **iteratiu** a l'arrel, a `.es`, a `gva.es` i a `cult.gva.es`. Cada un li dona una **referència** al següent, fins que l'**autoritatiu** dona la IP.
3. **⑩** El servidor local retorna la resposta final al client (i la guarda a la **cache**).

> **Resum:** el **client** pregunta de manera **recursiva**; els **servidors** es pregunten entre ells de manera **iterativa**.

### 7.3 Des d'on comença a buscar?

El servidor local **no sempre comença per l'arrel**. Comença pel servidor **més proper al domini buscat que ja conegui**. Exemple amb `www.jaumehuguet.xtec.cat`:

1. Coneix algun servidor de `jaumehuguet.xtec.cat`? → Li pregunta directament.
2. Si no, coneix algun servidor de `xtec.cat`? → Comença per aquí.
3. Si no, coneix algun servidor de `.cat`? → Comença per aquí.
4. Si no coneix res, **pregunta a un servidor arrel**. Des de l'arrel **sempre** es pot arribar a qualsevol domini.

---

## 8. La cache DNS

### 8.1 El problema: la saturació dels servidors arrel

Si **totes** les consultes del món comencessin pels servidors arrel, aquests **es saturarien**.

### 8.2 La solució: la cache

La **cache** és una memòria on el servidor DNS **guarda temporalment les respostes** que ja ha obtingut.

- Si algú torna a preguntar el mateix, el servidor **respon de seguida** des de la cache.
- Evita **sobrecarregar els servidors arrel** i els TLD.
- **Accelera** molt les resolucions.

Cada resposta es guarda només durant un temps limitat, anomenat **TTL** (*Time To Live*), que defineix l'administrador de la zona. Quan s'acaba el TTL, la dada s'esborra de la cache i caldrà tornar-la a consultar.

**Per tant, una resposta pot venir de:**

1. La **base de dades de la zona** (resposta autoritativa).
2. La **cache** (resposta no autoritativa).
3. Les respostes d'**altres servidors** consultats en el moment.

```bash
# Esborrar la cache DNS de l'equip
ipconfig /flushdns                 # Windows
sudo resolvectl flush-caches       # Linux (systemd-resolved)
```

---

## 9. La resolució inversa (in-addr.arpa)

### 9.1 Què és

La **resolució inversa** és el procés contrari a l'habitual: a partir d'una **adreça IP**, obtenir el **nom de domini**.

**Per a què serveix?**

- Molts protocols la necessiten per funcionar bé (per exemple, els **servidors de correu** rebutgen correus de servidors sense resolució inversa correcta).
- S'utilitza com a **mesura de seguretat** per verificar que una IP pertany realment a un domini.
- Fa més llegibles els **registres (logs)** dels servidors.

### 9.2 El domini especial `in-addr.arpa`

Per poder buscar IP dins de l'arbre DNS, es fa servir un domini especial: **`in-addr.arpa`**.

**Regla:** s'escriuen els **octets de la IP en ordre invers** i s'hi afegeix `.in-addr.arpa.`

| | IP / xarxa | Domini in-addr.arpa |
|---|---|---|
| **Host** | `192.168.1.24` | `24.1.168.192.in-addr.arpa.` |
| **Xarxa /24** | `172.16.32.0/24` | `32.16.172.in-addr.arpa.` |
| **Xarxa /16** | `10.20.0.0/16` | `20.10.in-addr.arpa.` |

**Per què a l'inrevés?** Perquè en una IP la part més **general** (la xarxa) és a l'**esquerra**, i en el DNS la part més general és a la **dreta**. Invertint-la, l'arbre queda ordenat igual que la resta del DNS.

```text
  IP:        192 . 168 .  1  . 24        (general → concret)
  DNS:        24 .  1  . 168 . 192 . in-addr.arpa.   (concret → general)
```

### 9.3 Exemple pràctic

```text
[root@localhost ~]# host -t NS ioc.cat
ioc.cat name server dns1.nominalia.com.
ioc.cat name server dns2.nominalia.com.

[root@localhost ~]# host dns1.nominalia.com
dns1.nominalia.com has address 62.97.103.68

[root@localhost ~]# host 62.97.103.68
68.103.97.62.in-addr.arpa domain name pointer dns1.nominalia.com.
```

1. Primer busquem quins són els servidors DNS de `ioc.cat`.
2. Obtenim la IP d'un d'ells.
3. Fem la resolució inversa de la IP i **confirmem** que correspon a `dns1.nominalia.com`.

---

## 10. El protocol DNS

El DNS és un protocol de **capa d'aplicació** que funciona amb un mecanisme de **pregunta/resposta**: els missatges són de tipus **query** (consulta) o **answer** (resposta).

| Transport | Quan s'utilitza |
|---|---|
| **UDP** (habitual) | Consultes i respostes petites (fins a **512 bytes** en el DNS clàssic). És el mode **per defecte**, perquè és ràpid. |
| **TCP** (si cal) | Quan la resposta és **massa gran** o en les **transferències de zona** entre primari i secundari. |
| **Port** | **53**, tant en UDP com en TCP. |

> **Per recordar:** **DNS = port 53**. Si un tallafoc bloqueja el port 53, l'equip no podrà resoldre noms (encara que tingui Internet!).

---

## 11. Estructura d'un missatge DNS

Tots els missatges DNS (consultes i respostes) tenen **cinc seccions**:

| Secció | Contingut |
|---|---|
| **HEADER** (capçalera) | Indica si és **consulta o resposta**, l'**identificador** (*id*), els **flags** i quants registres hi ha a cada secció. |
| **QUESTION** | La **pregunta**: quin nom i quin tipus de dada es demana. |
| **ANSWER** | La **resposta** a la pregunta. |
| **AUTHORITY** | Els **servidors autoritatius** del domini (pot estar buida). |
| **ADDITIONAL** | **Informació extra** útil, per exemple les IP dels servidors que surten a AUTHORITY. |

### 11.1 Exemple pràctic

```text
[root@portatil ~]# host -a uoc.es
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14091
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 2

;; QUESTION SECTION:
;uoc.es.                IN  ANY

;; ANSWER SECTION:
uoc.es.        82747    IN  NS  tibet.uoc.es.
uoc.es.        82747    IN  NS  nepal.uoc.es.

;; AUTHORITY SECTION:
uoc.es.        82747    IN  NS  nepal.uoc.es.
uoc.es.        82747    IN  NS  tibet.uoc.es.

;; ADDITIONAL SECTION:
nepal.uoc.es.  73649    IN  A   213.73.40.47
tibet.uoc.es.  76582    IN  A   213.73.40.45
```

**Com s'interpreta:**

- **HEADER:** `status: NOERROR` → tot correcte. Flags: `qr` = és una resposta, `rd` = s'ha demanat recursió, `ra` = el servidor accepta recursió.
- **QUESTION:** s'ha preguntat per `uoc.es`, qualsevol tipus de registre (`ANY`).
- **ANSWER:** els servidors de noms de `uoc.es` són `tibet` i `nepal`.
- **AUTHORITY:** els servidors autoritatius de la zona (els mateixos).
- **ADDITIONAL:** les IP d'aquests servidors, perquè no calgui fer una altra consulta.
- El número `82747` és el **TTL** en segons: el temps que la dada pot quedar a la cache.

> **Nota del professor:** com que no hi ha el flag `aa` (*authoritative answer*), sabem que aquesta resposta **no és autoritativa**: ve de la cache del servidor DNS de l'equip.

---

## 12. DDNS: DNS dinàmic

El **DDNS** (*Dynamic DNS*) permet que les dades del servidor DNS **s'actualitzin automàticament i en temps real**.

**Per a què serveix?**

- Perquè equips amb **IP dinàmica** puguin tenir un **nom fix**, encara que la seva IP canviï cada vegada que es connecten.

**Exemples d'ús:**

| Escenari | Com funciona |
|---|---|
| **Xarxa d'empresa** | El **servidor DHCP** dona una IP a un equip i **avisa el servidor DNS** perquè actualitzi el nom → IP. Així sempre es pot trobar `pc-joan.empresa.local`. |
| **Casa** | Tens una càmera o un servidor a casa i el teu proveïdor et canvia la IP pública. Un servei com **No-IP** o **DuckDNS** fa que `lamevacasa.duckdns.org` apunti sempre a la IP actual. |

> Això connecta el **RA1 (DHCP)** amb el **RA2 (DNS)**: tots dos serveis poden treballar junts.

---

## 13. DNSSEC: seguretat al DNS

### 13.1 El problema: el DNS original no és segur

El DNS es va dissenyar fa molts anys **sense pensar en la seguretat**. Un atacant podria fer-se passar per un servidor DNS i donar **respostes falses**, enviant-te a una web falsa (per exemple, una còpia de la web del teu banc).

### 13.2 L'atac *man-in-the-middle* (home al mig)

Un **man-in-the-middle** és un equip que **es posa enmig** d'una comunicació i es fa passar per l'altre extrem:

```text
  Client  ───▶  ATACANT  ───▶  Banc
          ◀───           ◀───
```

1. El client creu que es connecta al banc, però en realitat es connecta a l'atacant.
2. L'atacant reenvia el trànsit al banc, rep la resposta i la passa al client.
3. Mentrestant, pot **llegir i modificar** tot el que passa.

### 13.3 La solució: DNSSEC

**DNSSEC** (*Domain Name System Security Extensions*) és un conjunt d'extensions de seguretat que fa servir **signatures digitals** per garantir:

| Garantia | Explicació |
|---|---|
| **Autenticitat** | Que qui respon és el **servidor DNS legítim** i no un impostor. |
| **Integritat** | Que les dades **no s'han modificat** pel camí. |

> **Nota del professor:** a la presentació hi diu que DNSSEC està dissenyat per prevenir atacs de denegació de servei, però això **no és correcte**. DNSSEC protegeix contra la **falsificació de respostes** (*DNS spoofing* o enverinament de cache) i atacs *man-in-the-middle*. **No xifra** les consultes ni protegeix contra atacs de denegació de servei (DoS).

---

## 14. Resum

| Concepte | Idea clau |
|---|---|
| **Zona** | Part de l'espai de noms gestionada per uns servidors concrets, que hi tenen autoritat. Domini ≠ zona. |
| **Delegació** | Cedir l'autoritat d'un subdomini a altres servidors. La zona pare només guarda un punter. |
| **Primari / secundari** | Mínim 2 servidors per zona. El primari té els originals; el secundari en rep còpia per transferència de zona. |
| **Resolver** | Part client del DNS: biblioteques del sistema que fan les consultes. |
| **Recursiu / iteratiu** | El client pregunta de manera recursiva; els servidors entre ells, de manera iterativa. |
| **Cache** | Guarda respostes durant el TTL. Evita saturar l'arrel i accelera les consultes. |
| **Resolució inversa** | IP → nom, amb el domini `in-addr.arpa` i els octets invertits. |
| **Protocol** | Port 53. UDP per defecte; TCP per a respostes grans i transferències de zona. |
| **Seguretat** | DDNS = actualitzacions automàtiques. DNSSEC = signatures per evitar respostes falses. |

---

## 15. Preguntes de repàs

1. Què és una zona DNS? Quina diferència hi ha entre un domini i una zona?
2. En l'exemple de les quatre zones, quins nodes pertanyen a la zona `C.A`? I al domini `C.A`? Per què no coincideixen?
3. Què vol dir delegar una zona? Què conserva la zona pare després de la delegació?
4. Per què l'estàndard DNS demana almenys dos servidors autoritatius per zona?
5. Quina diferència hi ha entre un servidor primari i un de secundari? Com obté les dades el secundari?
6. Què és el resolver? És un programa independent?
7. Quina diferència hi ha entre una resposta autoritativa i una de no autoritativa?
8. Explica amb les teves paraules la diferència entre la resolució recursiva i la iterativa.
9. Descriu els passos per resoldre `www.cult.gva.es` si el servidor local no té res a la cache.
10. Per què és tan important la cache DNS? Què és el TTL?
11. Escriu el domini `in-addr.arpa` corresponent a la IP `10.0.5.200` i a la xarxa `192.168.50.0/24`.
12. Quin port i quins protocols de transport fa servir el DNS? Quan s'utilitza TCP?
13. Quines són les cinc seccions d'un missatge DNS? Què conté la secció ADDITIONAL?
14. Què és el DDNS? Posa'n un exemple relacionat amb el DHCP.
15. Contra quin tipus d'atac protegeix DNSSEC? Com ho fa?
