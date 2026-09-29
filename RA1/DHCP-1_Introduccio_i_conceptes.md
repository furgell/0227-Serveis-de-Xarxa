# Serveis de Xarxa · RA1 · DHCP (Part 1)

## Introducció al servei DHCP i al protocol

**Cicle:** 2n SMX (Sistemes Microinformàtics i Xarxes)
**Mòdul:** Serveis de Xarxa
**Resultat d'aprenentatge 1:** Instal·lació i configuració d'un servei DHCP

---

## Índex

1. [El servei DHCP](#1-el-servei-dhcp)
2. [Configuració d'un equip de xarxa](#2-configuració-dun-equip-de-xarxa)
3. [Tipus d'assignacions](#3-tipus-dassignacions)
4. [El protocol DHCP](#4-el-protocol-dhcp)
5. [Instal·lació i configuració bàsica (visió general)](#5-installació-i-configuració-bàsica-visió-general)
6. [Configuració del client](#6-configuració-del-client)
7. [Comprovació del funcionament](#7-comprovació-del-funcionament)
8. [Documentació de suport a l'usuari](#8-documentació-de-suport-a-lusuari)
9. [Resum](#9-resum)
10. [Preguntes de repàs](#10-preguntes-de-repàs)

---

## 1. El servei DHCP

### 1.1 Què és el DHCP?

**DHCP** són les sigles de **Dynamic Host Configuration Protocol** (*protocol de configuració dinàmica d'equips*).

És un servei de xarxa que permet configurar **de manera totalment automàtica** els paràmetres de xarxa dels equips. Els principals són:

| Paràmetre | Per a què serveix |
|---|---|
| **Adreça IP** | Identifica de manera única cada equip a la xarxa. |
| **Màscara de xarxa** | Determina la xarxa o subxarxa a la qual pertany l'equip. |
| **Passarel·la (gateway)** | Porta d'enllaç per defecte per sortir cap a Internet. |
| **Altres opcions** | Servidors DNS, nom d'host, fitxers d'arrencada, etc. |

### 1.2 Una analogia per entendre'l

Quan un equip client s'engega, **"crida" per la xarxa**:

> «Hi ha algú? Qui sóc jo?»

El **servidor DHCP** respon donant al client tota la informació necessària perquè sàpiga **qui és** i **com s'ha de configurar** a la xarxa.

### 1.3 Identificació dels equips de xarxa

L'administrador de xarxa ha de configurar tots els equips: servidors, clients, concentradors (*hubs*), encaminadors (*routers*), etc. Cada equip necessita:

- **Adreça IP i màscara:** cada equip s'identifica amb la seva adreça IP i la màscara de xarxa.
- **Accés a Internet:** normalment mitjançant una porta d'enllaç.
- **Nom de domini:** usuaris i serveis identifiquen els equips pel nom, que és més fàcil de recordar que l'adreça IP.

### 1.4 Configuració centralitzada

El DHCP **simplifica l'administració** perquè la configuració és:

- **Centralitzada:** es fa en un únic punt (el servidor).
- **Dinàmica:** el servidor assigna els valors als clients quan els necessiten.
- **Temporal:** les assignacions es fan per **períodes de temps finits** (concessions o *leases*). Quan acaba el període, cal renovar-les.

En comptes de configurar un per un els equips amb valors estàtics, el servidor DHCP va assignant als clients els valors que els corresponen.

### 1.5 Períodes de concessió d'adreces IP

La durada de la concessió depèn de les necessitats del client i del servidor:

| Escenari | Durada típica de la concessió | Motiu |
|---|---|---|
| **Biblioteca / Wi-Fi públic** | Minuts | Connexions de curta durada. |
| **Usuari domèstic (ISP)** | Hores | La IP dinàmica del proveïdor d'Internet. |
| **Xarxa corporativa** | Dies | Els equips d'empresa són estables i romanen connectats molt temps. |

---

## 2. Configuració d'un equip de xarxa

### 2.1 Paràmetres mínims

Qualsevol equip d'una xarxa necessita uns **paràmetres mínims** per tenir connectivitat:

1. **Adreça IP:** identifica l'equip de manera única a la xarxa.
2. **Màscara de xarxa:** permet saber en quina xarxa o subxarxa es troba l'equip.
3. **Porta d'enllaç (gateway):** necessària per accedir fora de la xarxa pròpia.

> **Important:** amb l'adreça IP i la màscara n'hi ha prou per tenir connectivitat **dins la xarxa local**. La porta d'enllaç només cal per **sortir a l'exterior** (Internet, altres xarxes).

### 2.2 Exemple real de configuració domèstica

Sortida de `ipconfig /all` en un Windows d'una xarxa domèstica:

```text
Dirección IP. . . . . . . . . . : 192.168.1.33
Máscara de subred . . . . . . . : 255.255.255.0
Puerta de enlace predeterminada : 192.168.1.1
Servidor DHCP . . . . . . . . . : 192.168.1.1
Servidores DNS  . . . . . . . . : 80.58.61.250
                                  80.58.61.254
```

Observa que el servidor DHCP és l'encaminador de casa (`192.168.1.1`), que també és la porta d'enllaç.

A part d'això, els equips poden necessitar **paràmetres addicionals**: nom d'host, servidors DNS, un fitxer d'iniciació per descarregar, etc.

### 2.3 El problema de la configuració manual

Configurar manualment implica:

- Anar **equip per equip**.
- Risc d'**errades en teclejar** adreces i màscares.
- Si canvia l'estructura de la xarxa, cal **tornar a configurar tots els equips** afectats.

Tant si la xarxa té pocs equips com molts, cal una solució per **automatitzar la configuració de manera centralitzada**.

> **Configuració estàtica vs. dinàmica:** fins i tot amb accés remot (Telnet o SSH) cal modificar la configuració equip per equip. El DHCP elimina aquest problema.

---

## 3. Tipus d'assignacions

### 3.1 IP estàtica vs. IP dinàmica

| | **IP estàtica** | **IP dinàmica (DHCP)** |
|---|---|---|
| **Com es configura** | Manualment, equip per equip. | Automàticament, mitjançant un servidor DHCP. |
| **Adreça** | Sempre la mateixa, definida per l'administrador. | La rep del servidor. |
| **Requisits** | Cap. | Estructura **client-servidor**. |

### 3.2 Tipus d'assignació dinàmica

**a) Assignació dinàmica d'interval**

- El servidor disposa d'un **interval (pool) d'adreces** que pot assignar.
- El client **no sap quina IP tindrà** i no es pot predir.
- A cada nova assignació, l'adreça pot ser **diferent**.

**b) Assignació fixa (reserva)**

- El servidor **sempre assigna la mateixa adreça** al mateix client.
- Identifica el client de manera inequívoca per la seva **adreça MAC**.
- Consulta una **taula de correspondències MAC ↔ IP**.

### 3.3 Avantatges del DHCP

| Avantatge | Descripció |
|---|---|
| **Evita conflictes** | Elimina les adreces IP repetides i errònies. |
| **Automatització** | Configuració automàtica sense intervenció manual. |
| **Estalvi de temps** | Administració centralitzada per a tota la xarxa. |
| **Concessions temporals** | Assignació per períodes finits, renovables automàticament. |

---

## 4. El protocol DHCP

### 4.1 Què són els RFC?

El protocol DHCP està descrit en un document oficial anomenat **RFC** (*Request for Comments*).

- Els RFC són **memoràndums** sobre noves investigacions, innovacions i metodologies relacionades amb les tecnologies d'Internet.
- Quan els publica l'**IETF** (*Internet Engineering Task Force*), defineixen a escala mundial els protocols i les seves revisions.
- El protocol DHCP ha evolucionat al llarg dels anys per adaptar-se a les necessitats de cada moment.

### 4.2 Arquitectura del protocol

- Es basa en el model **client-servidor**.
- Utilitza com a transport el protocol **UDP** (pila TCP/IP).
- Els **ports** són estàndard (definits per l'RFC):

| Component | Port UDP |
|---|---|
| **Servidor DHCP** (rep les peticions) | **67** |
| **Client DHCP** (rep les respostes) | **68** |

```text
   CLIENT (port 68)                          SERVIDOR (port 67)
        │ ───────── Client → Servidor ─────────▶ │  (peticions al port 67)
        │ ◀──────── Servidor → Client ────────── │  (respostes al port 68)
```

### 4.3 Evolució del protocol: de BOOTP a DHCP

La configuració dinàmica d'equips va començar amb **BOOTP** (*Bootstrap Protocol*), un protocol bàsic que permetia definir l'adreça IP, la màscara i la passarel·la per defecte durant l'arrencada. Es feia servir originàriament per a **estacions de treball sense disc**.

| Any | RFC | Fita |
|---|---|---|
| **1985** | RFC 951 | Definició del protocol **BOOTP** (precursor del DHCP). |
| **1993** (octubre) | RFC 1531 | Primera especificació del protocol **DHCP**. |
| **1997** (març) | RFC 2131 | Base del **DHCP actual per a IPv4**. |
| — | RFC 2132 | Conjunt d'**opcions** de configuració del DHCP. |
| — | RFC 3315 | Especificació del **DHCP per a IPv6** (DHCPv6). |

> **Nota per ampliar:** l'RFC 3315 (DHCPv6) va ser substituït posteriorment per l'RFC 8415. L'RFC 1531 va ser substituït per l'RFC 2131. Per a IPv4, l'RFC 2131 continua sent la referència.

---

## 5. Instal·lació i configuració bàsica (visió general)

> En el document 3 veurem la instal·lació i configuració amb molt més detall. Aquí en fem una primera visió.

### 5.1 Seqüència lògica d'instal·lació

```text
  1. Instal·lar  →  2. Configurar el pool  →  3. Definir opcions  →  4. Iniciar el servei
```

### 5.2 Comandes d'instal·lació (Linux / Ubuntu)

```bash
# Actualitzar els repositoris
sudo apt update

# Instal·lar el servidor DHCP
sudo apt install isc-dhcp-server

# Comprovar l'estat del servei
systemctl status isc-dhcp-server

# Habilitar el servei a l'inici del sistema
sudo systemctl enable isc-dhcp-server
```

### 5.3 Passos de configuració del servidor

1. **Definir l'interval d'adreces (pool):** rang d'IP disponibles per assignar dinàmicament.
2. **Configurar les opcions globals:** màscara, porta d'enllaç, DNS i altres paràmetres comuns.
3. **Definir reserves (assignació fixa):** associar adreces MAC a IP fixes.
4. **Establir el temps de concessió:** el *lease time* adequat segons el tipus de xarxa.

### 5.4 Exemple de configuració bàsica

```bash
# Editar el fitxer de configuració principal
sudo nano /etc/dhcp/dhcpd.conf
```

```conf
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  default-lease-time 600;
  max-lease-time 7200;
}
```

Cal indicar també **per quina interfície de xarxa** escoltarà el servidor:

```bash
sudo nano /etc/default/isc-dhcp-server
```

```conf
INTERFACESv4="eth0"
```

Finalment, reiniciem el servei per aplicar els canvis:

```bash
sudo systemctl restart isc-dhcp-server
```

---

## 6. Configuració del client

### 6.1 Activar DHCP al client

- **Windows:** s'activa a les **propietats de la targeta de xarxa** (obtenir IP automàticament).
- **Linux:** es configura al fitxer de xarxa o amb eines com `dhclient`.

Un cop activat, el client **envia automàticament una petició DHCP en arrencar** i obté tots els paràmetres sense cap intervenció de l'usuari.

### 6.2 Client DHCP a Linux

```bash
# Activar DHCP en una interfície (temporal)
sudo dhclient eth0

# Alliberar l'adreça IP actual
sudo dhclient -r eth0

# Renovar l'adreça IP
sudo dhclient eth0

# Veure la configuració IP actual
ip addr show eth0
```

**Configuració permanent amb Netplan (Ubuntu):**

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
```

```bash
# Aplicar la configuració Netplan
sudo netplan apply
```

---

## 7. Comprovació del funcionament

Un cop instal·lat i configurat el servei, cal **verificar que funciona correctament**.

### 7.1 Eines de comprovació

| Eina | Sistema | Què mostra |
|---|---|---|
| `ipconfig /all` | Windows | Configuració IP actual i el servidor DHCP que l'ha assignada. |
| `ip addr` / `ifconfig` | Linux | Adreces IP assignades a les interfícies. |
| **Registres (logs) del servidor** | Servidor | Concessions actives i peticions rebudes. |

### 7.2 Comandes Linux útils

**Gestió del client DHCP:**

```bash
# Sol·licitar adreça IP al servidor DHCP
dhclient eth0

# Alliberar l'adreça IP actual
dhclient -r eth0

# Mostrar les adreces IP assignades
ip addr show

# Veure les concessions del client
cat /var/lib/dhcp/dhclient.leases
```

**Gestió del servidor DHCP:**

```bash
# Comprovar l'estat del servidor DHCP
systemctl status isc-dhcp-server

# Veure els registres del servidor DHCP
journalctl -u isc-dhcp-server

# Veure la configuració del servidor
cat /etc/dhcp/dhcpd.conf
```

### 7.3 Verificació completa

La verificació completa comprova **dos costats**:

- **Client:** rep la configuració correcta.
- **Servidor:** registra les concessions i **no hi ha conflictes d'adreces**.

Flux de verificació: **Sol·licitud del client → Assignació del servidor → Confirmació del client → Revisió de l'administrador.**

---

## 8. Documentació de suport a l'usuari

Una part fonamental de la posada en marxa d'un servei és **documentar-lo bé**. La documentació ha d'incloure:

| Apartat | Contingut |
|---|---|
| **Configuració del servidor** | Intervals d'adreces, opcions configurades, temps de concessió i reserves fixes. |
| **Procediments de comprovació** | Passos per verificar el funcionament i eines de diagnòstic disponibles. |
| **Guia per a l'usuari** | Instruccions senzilles per activar DHCP al client i resoldre problemes bàsics de connectivitat. |

---

## 9. Resum

| Concepte | Idea clau |
|---|---|
| **Definició** | Protocol de configuració dinàmica d'equips (RFC 2131). Client-servidor sobre UDP (ports 67/68). |
| **Assignació** | Dinàmica d'interval (adreça variable) o fixa per MAC. Concessions per períodes finits. |
| **Avantatges** | Evita conflictes d'IP, centralitza l'administració i automatitza la configuració. |
| **Evolució** | De BOOTP (1985) a DHCP (1993–1997). RFC 3315 per a IPv6. Estàndards publicats per l'IETF. |

> El DHCP és un servei fonamental en qualsevol xarxa moderna. La seva correcta **instal·lació, configuració i documentació** és essencial per a l'administrador de xarxa.

---

## 10. Preguntes de repàs

1. Què significa DHCP i quins paràmetres configura automàticament?
2. Quins són els tres paràmetres mínims de configuració d'un equip? Quins són suficients per comunicar-se només dins la xarxa local?
3. Quina diferència hi ha entre una IP estàtica i una IP dinàmica?
4. Explica la diferència entre l'assignació dinàmica d'interval i l'assignació fixa. Com s'identifica el client en la segona?
5. Enumera quatre avantatges del DHCP.
6. Quins ports UDP utilitzen el servidor i el client DHCP?
7. Quin protocol va precedir el DHCP? Per a què s'utilitzava?
8. Quin RFC és la base del DHCP actual per a IPv4?
9. Quina comanda permet veure la configuració IP en Windows? I en Linux?
10. Quins elements ha de recollir la documentació d'un servei DHCP?
