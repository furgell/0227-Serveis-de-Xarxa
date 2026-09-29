# Serveis de Xarxa · RA1 · DHCP (Part 2)

## El funcionament del protocol DHCP

**Cicle:** 2n SMX (Sistemes Microinformàtics i Xarxes)
**Mòdul:** Serveis de Xarxa
**Resultat d'aprenentatge 1:** Instal·lació i configuració d'un servei DHCP

---

## Índex

1. [El model funcional del protocol](#1-el-model-funcional-del-protocol)
2. [Tipus de paquets DHCP](#2-tipus-de-paquets-dhcp)
3. [Renovació d'una concessió](#3-renovació-duna-concessió)
4. [Seguretat](#4-seguretat)
5. [Configuració: conceptes bàsics](#5-configuració-conceptes-bàsics)
6. [DHCP: un servei client/servidor](#6-dhcp-un-servei-clientservidor)
7. [El client DHCP en detall](#7-el-client-dhcp-en-detall)
8. [El servidor DHCP](#8-el-servidor-dhcp)
9. [Resum](#9-resum)
10. [Preguntes de repàs](#10-preguntes-de-repàs)

---

## 1. El model funcional del protocol

El protocol DHCP descriu el **diàleg entre client i servidor** per concedir configuracions IP.

- El client **sol·licita** una configuració al servidor.
- S'inicia un procés de **negociació** que, si tot va bé, acaba amb la **concessió** d'una adreça IP.
- La concessió és per un **període de temps establert pel servidor**. Quan acaba, el client ha de **renegociar-la** amb un procés similar.

### 1.1 Les quatre fases de la negociació (DORA)

| Fase | Missatge | Qui l'envia | Què fa |
|---|---|---|---|
| **1** | **D**iscovery | Client | Sol·licita una adreça IP (configuració de xarxa). |
| **2** | **O**ffer | Servidor | Ofereix una IP disponible del seu interval dinàmic. |
| **3** | **R**equest | Client | Accepta l'oferta i ho comunica per difusió (*broadcast*). |
| **4** | **A**cknowledge | Servidor | Confirma la concessió per un temps limitat. |

> **Truc per recordar-ho:** les inicials formen la paraula **DORA** (Discover, Offer, Request, Ack).

### 1.2 Diagrama de seqüència

```text
     CLIENT                                             SERVIDOR DHCP
       │                                                     │
       │ ── 1. DHCP DISCOVER  (broadcast) ─────────────────▶ │  "Hi ha algun servidor DHCP?"
       │                                                     │
       │ ◀───────────────── 2. DHCP OFFER  (unicast) ─────── │  "Et puc oferir aquesta IP"
       │                                                     │
       │ ── 3. DHCP REQUEST   (broadcast) ─────────────────▶ │  "Accepto aquesta oferta"
       │                                                     │
       │ ◀───────────────── 4. DHCP ACK  (unicast) ────────── │  "Concessió confirmada"
       │                                                     │
       ▼  El client ja pot utilitzar la IP durant el temps de concessió
```

| Missatge | Tipus d'enviament | Direcció |
|---|---|---|
| Discovery | Broadcast | Client → Servidor |
| Offer | Unicast | Servidor → Client |
| Request | Broadcast | Client → Servidor |
| Acknowledge | Unicast | Servidor → Client |

---

## 2. Tipus de paquets DHCP

El protocol defineix **set tipus de paquets**, cadascun amb un rol específic en la negociació i la gestió de les concessions:

| Paquet | Emissor | Funció resumida |
|---|---|---|
| **DHCP Discover** | Client | Primera petició de configuració (broadcast). |
| **DHCP Offer** | Servidor | Oferta d'una IP i paràmetres. |
| **DHCP Request** | Client | Acceptació de l'oferta o petició de renovació. |
| **DHCP Ack / Nack** | Servidor | Confirmació o denegació de la concessió. |
| **DHCP Decline** | Client | Rebuig d'una IP. |
| **DHCP Release** | Client | Alliberament de la IP. |
| **DHCP Information** | Client | Demana paràmetres addicionals. |

### 2.1 DHCP Discover

- És **el primer paquet** que s'envia. L'envia el client quan s'acaba d'inicialitzar i vol una configuració dinàmica.
- El client **no sap a quina xarxa pertany** (no té IP ni màscara) ni quins servidors DHCP hi ha.
- Per això genera un paquet de **difusió (broadcast)** destinat a tots els equips de la xarxa, sol·licitant una configuració IP.

### 2.2 DHCP Offer

Quan el servidor rep un Discover:

1. Mira les **adreces lliures** del seu pool dinàmic.
2. **Ofereix** una d'elles al client.
3. **Anota** cada concessió en un fitxer de registre.

**Contingut del paquet Offer:**

- IP i MAC d'origen del servidor.
- Adreça IP oferta al client.
- Durada de la concessió (*lease*).
- Porta d'enllaç per defecte.
- Servidors DNS.

S'envia per **unicast** directament al client. Tota concessió és per un període determinat i cal renovar-la quan acaba.

> **Matís tècnic:** com que el client encara no té IP configurada, en la pràctica moltes implementacions envien l'Offer amb la MAC del client com a destinació física. Segons el bit *broadcast flag* del Discover, algunes també l'envien per broadcast. Aquí seguim el model de l'RFC que hem vist a classe.

### 2.3 DHCP Request

- Si el client **accepta l'oferta**, ho comunica al servidor amb un DHCP Request enviat per **broadcast**.

**Per què broadcast?**

- Fa **públic a tota la xarxa** quina oferta accepta el client.
- Els **altres servidors DHCP** entenen que la seva oferta ha estat **rebutjada**.
- El client encara **no disposa de l'adreça IP definitiva**.

### 2.4 DHCP Ack i DHCP Nack

| | **DHCP ACK ✓** | **DHCP NACK ✗** |
|---|---|---|
| **Significat** | Autorització de la concessió. | Denegació de la concessió. |
| **Què passa** | A partir d'aquí el client **ja pot utilitzar la IP**. | El client ha de **reiniciar tot el procés des del Discover**. |
| **Contingut** | Durada de la concessió i dades per gestionar-ne l'expiració. | — |
| **Enviament** | Unicast a la MAC del client. | — |
| **Quan es produeix** | Tot va bé. | La IP sol·licitada ja està en ús, és fora d'interval, etc. Pot indicar un **equip mal configurat** a la xarxa. |

### 2.5 DHCP Decline, Release i Information

| Paquet | Descripció |
|---|---|
| **DHCP Decline** | El client **rebutja** l'adreça si detecta que la IP **ja està en ús** o no li convé (p. ex., en una renovació rep una IP diferent de la que fa servir). |
| **DHCP Release** | Quan el client **ja no necessita la IP**, l'allibera. El servidor la **retorna al pool**. Sovint no s'emet, perquè l'equip s'apaga sense temps d'alliberar-la. |
| **DHCP Information** | El client, **ja configurat**, demana més informació (WINS, NetBIOS, hostname, etc.). El servidor respon amb els paràmetres demanats. |

---

## 3. Renovació d'una concessió

Hi ha dues situacions:

| Situació | Procés |
|---|---|
| **El client vol una IP nova** | Procés complet de 4 fases: **Discovery → Offer → Request → Ack**. |
| **El client vol continuar amb la mateixa IP** | Renovació simplificada: envia directament un **DHCP Request** i el servidor respon amb **ACK** o **NACK**. |

**NACK en renovació:** si el servidor no pot concedir la IP sol·licitada (en ús, fora d'interval, etc.), envia un DHCP NACK i el client ha de tornar a començar.

> **Per ampliar:** en una implementació típica, el client intenta renovar quan ha passat aproximadament el **50 %** del temps de concessió (T1) i, si no obté resposta, es posa a **rebre de qualsevol servidor** cap al **87,5 %** (T2). Si la concessió expira sense renovar-se, ha de reiniciar el procés des del Discover.

### 3.1 Renovar la concessió per línia de comandes

```bash
# Alliberar i renovar la concessió (dhclient)
dhclient -r eth0 && dhclient eth0

# Baixar i pujar la connexió amb NetworkManager
nmcli connection down eth0 && nmcli connection up eth0

# Netejar la IP assignada a la interfície
ip addr flush dev eth0
```

En Windows:

```cmd
ipconfig /release
ipconfig /renew
```

---

## 4. Seguretat

### 4.1 Atacs al funcionament de DHCP

**Atac per inundació (DHCP starvation)**

- Un client **inunda el servidor** amb peticions DHCP Discover **fingint ser clients diferents** (falsificant la MAC).
- **Objectiu:** esgotar les IP disponibles i sobrecarregar el servidor fins a bloquejar-lo.

**Servidor DHCP fals (rogue DHCP)**

- Un servidor atacant **respon a les peticions broadcast** del client amb una configuració falsa.
- Exemple: un **servidor DNS maliciós** que redirigeix el tràfic bancari a màquines fraudulentes.

### 4.2 Mesures de seguretat

- El protocol permet utilitzar mecanismes d'**autenticació i xifratge** per garantir la legitimitat dels missatges entre client i servidor.

> **Per ampliar:** en xarxes reals també es fan servir mesures als commutadors, com **DHCP Snooping** (només es permeten respostes de servidors DHCP a ports de confiança) i **Port Security** (limitar el nombre de MAC per port).

### 4.3 Conflictes d'adreces IP

Alguns problemes habituals:

- **Dues màquines amb la mateixa IP** per una mala configuració del servidor DHCP.
- Un client amb **IP estàtica coincident** amb una IP del pool dinàmic.
- **Configuració local del client sobreescrita** pels paràmetres DHCP (DNS, porta d'enllaç, etc.).

> La configuració dinàmica pot **sobreescriure paràmetres locals** del client. Això és precisament l'objectiu del DHCP, però pot sorprendre administradors poc experimentats.

**Bona pràctica:** les IP estàtiques dels equips (servidors, impressores...) s'han de posar **fora de l'interval dinàmic** o bé com a **reserves** al servidor DHCP.

---

## 5. Configuració: conceptes bàsics

| Concepte | Definició |
|---|---|
| **Interval (range / pool)** | Conjunt d'adreces dinàmiques disponibles per assignar. S'agrupen per **subxarxa**. Una mateixa subxarxa pot tenir **diversos intervals**. |
| **Exclusions** | Adreces IP que **no s'ofereixen dinàmicament**. No formen part de cap interval del servidor. |
| **Concessions (leases)** | Assignació d'una IP i paràmetres de xarxa a un client per un **període de temps finit**. Cal renegociar-les en finalitzar. |
| **Reserves** | IP assignades via DHCP però de manera **fixa** a un host determinat per la seva **MAC**. Si el host s'apaga, la IP **no es pot usar per a altres**. |

### 5.1 Exemple: intervals per subxarxa

```conf
subnet 140.220.191.0 netmask 255.255.255.0 {
    range 140.220.191.150 140.220.191.249;
}

subnet 239.252.197.0 netmask 255.255.255.0 {
    range 239.252.197.10  239.252.197.107;
    range 239.252.197.113 239.252.197.250;
}
```

Fixa't que la segona subxarxa té **dos intervals**. Les adreces entre `.108` i `.112` **queden excloses** (no s'ofereixen dinàmicament).

### 5.2 Exemple: reserva per adreça MAC

```conf
subnet 140.220.191.0 netmask 255.255.255.0 {
    host iocserver {
        hardware ethernet 08:00:2b:4c:59:23;
        fixed-address 140.220.191.1;
    }
    range 140.220.191.150 140.220.191.249;
}
```

Les reserves permeten assignar **sempre la mateixa IP** a un host concret identificat per la seva MAC, combinant la **flexibilitat del DHCP** amb l'**estabilitat d'una IP fixa**.

### 5.3 Gestió de les concessions (leases)

- **Registre bilateral:** client i servidor anoten les concessions. El client, la que rep; el servidor, **totes** les que concedeix.
- **Renovació o revocació:** en finalitzar, el servidor pot **revocar-la o ampliar-la**. El client pot **renunciar-hi** o iniciar un **diàleg abreujat** per renovar-la.
- **Repetició d'assignació:** client i servidor consulten les concessions anteriors per **repetir, si és possible, la mateixa IP**.

**Consultar concessions per línia de comandes:**

```bash
# Concessions del servidor
cat /var/lib/dhcpd/dhcpd.leases        # RHEL/Fedora
cat /var/lib/dhcp/dhcpd.leases         # Debian/Ubuntu

# Concessions del client
cat /var/lib/dhcp/dhclient.leases

# Veure l'estat de cada concessió
grep "binding state" /var/lib/dhcpd/dhcpd.leases

# Comptar concessions
grep "lease" /var/lib/dhcpd/dhcpd.leases | wc -l
```

---

## 6. DHCP: un servei client/servidor

| **Servidor DHCP** | **Client DHCP** |
|---|---|
| Equip amb el **programa servidor en execució**. | Equip que **sol·licita** la IP i altres paràmetres al servidor, en lloc de tenir-los definits localment. |
| Atén les peticions, ofereix configuració i **registra totes les IP concedides** i les accions fetes. | Un equip pot exercir **les dues funcions simultàniament**. |

**Exemple pràctic: l'encaminador domèstic**

- Obté una **IP pública dinàmica de l'ISP** → actua com a **client**.
- Proporciona **IP privades dinàmiques** als ordinadors de casa → actua com a **servidor**.

---

## 7. El client DHCP en detall

### 7.1 Components

- **Dimoni (daemon) client:** ha d'estar en funcionament. Gestiona les tasques DHCP: **negocia** (Discovery, Request), porta el **registre de leases** i **es reactiva** automàticament quan cal renegociar.
- **Fitxer de leases:** el registre de concessions rebudes permet al client **tornar a demanar la mateixa IP**. Un cop rebuda la concessió, el dimoni queda "adormit" fins a la propera renovació.

```bash
# Veure estat del dimoni
systemctl status NetworkManager

# Veure concessions rebudes
cat /var/lib/dhcp/dhclient.leases

# Veure logs DHCP del client
journalctl -u NetworkManager | grep -i dhcp

# Mode verbose per veure la negociació en temps real
dhclient -v eth0
```

### 7.2 Clients DHCP: Windows vs. GNU/Linux

| | **Windows** | **GNU/Linux** |
|---|---|---|
| **Configuració** | Tendeix a la **configuració gràfica** amb finestres. | Mitjançant **fitxers de text** o opcions a l'ordre d'execució. |
| **Gestió DHCP** | S'executa internament, **d'amagat** de l'usuari. | Comportament **altament configurable**. |
| **Fitxer de leases** | Accessible però poc visible. | Accessible i detallat. |

En tots dos sistemes es pot configurar el client per definir la informació a demanar, la informació a proporcionar al servidor i les opcions per defecte.

```bash
# Sol·licitar IP per DHCP
dhclient eth0

# Alliberar la concessió
dhclient -r eth0

# Veure el fitxer de leases
cat /var/lib/dhcp/dhclient.leases

# Veure la IP assignada
ip addr show eth0

# Veure detalls de xarxa amb NetworkManager
nmcli device show eth0
```

---

## 8. El servidor DHCP

### 8.1 Xarxa bàsica vs. xarxa complexa

| **Xarxa corporativa bàsica** | **Xarxa corporativa complexa** |
|---|---|
| Un **únic servidor DHCP** dona servei a tots els equips, en una o diverses subxarxes amb connectivitat directa. | **Subxarxes segmentades amb tallafocs.** |
| Equivalent a la xarxa domèstica amb l'encaminador de l'ISP. | Dues opcions: **(1)** un únic servidor amb tallafocs que deixin passar els paquets DHCP, o **(2)** un **servidor per subxarxa** o grup de subxarxes. |

> **Important:** el servidor DHCP utilitza una **IP estàtica** definida per l'administrador, perquè ha d'estar **sempre disponible a la mateixa adreça**.

### 8.2 Tipus de programes servidor DHCP

| Tipus | Descripció |
|---|---|
| **Mode text** | Configuració i gestió mitjançant **ordres i fitxers de text**. Típic d'entorns Unix/Linux. |
| **Mode gràfic (GUI)** | **Interfície visual**. Més accessible per a administradors menys experimentats. |
| **Mode "màgic"** | El servidor funciona **de manera transparent**, sense que l'administrador vegi què fa internament. |

### 8.3 Tasques d'administració del servidor

| # | Tasca | Descripció |
|---|---|---|
| 1 | **Observar la configuració** | Consultar intervals, reserves, exclusions i paràmetres actius. |
| 2 | **Activar / Aturar el servei** | Gestionar l'estat del dimoni. |
| 3 | **Modificar la configuració** | Actualitzar intervals, reserves, durades de concessió, etc. |
| 4 | **Monitorar els logs** | Detectar problemes, atacs o comportaments anòmals. |
| 5 | **Instal·lar / Desinstal·lar** | Gestionar l'aplicació al sistema. |

```bash
# Observar l'estat del servei
systemctl status isc-dhcp-server

# Activar, aturar i reiniciar el servei
systemctl start isc-dhcp-server
systemctl stop isc-dhcp-server
systemctl restart isc-dhcp-server

# Modificar la configuració
nano /etc/dhcp/dhcpd.conf

# Monitorar els logs en temps real
journalctl -u isc-dhcp-server -f

# Instal·lar / Desinstal·lar el servidor DHCP
apt install isc-dhcp-server
apt remove isc-dhcp-server
```

### 8.4 Funcionament intern del servidor

1. **Escolta al port 67:** el servidor està sempre engegat esperant peticions.
2. **Processament simultani:** en rebre una petició, la processa i posa en marxa el mecanisme DHCP corresponent, **mentre continua escoltant** noves peticions.
3. **Persistència dels logs:** els fitxers de registre mantenen la informació de concessions **fins i tot si el servei s'atura o el servidor s'apaga**.

```bash
# Verificar que el servidor escolta al port 67
ss -ulnp | grep 67

# Veure totes les concessions actives
cat /var/lib/dhcpd/dhcpd.leases

# Monitorar logs en temps real
tail -f /var/log/syslog | grep dhcp

# Verificar la sintaxi del fitxer de configuració
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

> **Consell pràctic:** executa sempre `dhcpd -t` **abans de reiniciar** el servei. Un error de sintaxi al `dhcpd.conf` impedeix que el servidor arrenqui.

---

## 9. Resum

| Concepte | Idea clau |
|---|---|
| **Model funcional** | Discover → Offer → Request → Ack. Quatre fases de negociació client-servidor. |
| **Tipus de paquets** | 7 tipus: Discover, Offer, Request, Ack/Nack, Decline, Release, Information. |
| **Seguretat** | Atacs per inundació i servidors falsos. Solució: autenticació i xifratge. |
| **Configuració** | Intervals, exclusions, concessions (leases) i reserves per MAC. |
| **Arquitectura** | Servei client/servidor. El servidor usa IP estàtica i escolta al port 67. |

El procés complet de negociació DHCP garanteix que **cada client obtingui una configuració vàlida i única**. Les concessions són temporals i cal renovar-les periòdicament.

---

## 10. Preguntes de repàs

1. Explica les quatre fases de la negociació DHCP i qui envia cada missatge.
2. Per què el DHCP Discover s'envia per broadcast?
3. Per què el DHCP Request també s'envia per broadcast? Què entenen els altres servidors?
4. Quina diferència hi ha entre un DHCP ACK i un DHCP NACK? Què ha de fer el client quan rep un NACK?
5. Quan s'utilitza un DHCP Decline? I un DHCP Release?
6. Com es renova una concessió si el client vol mantenir la mateixa IP?
7. Descriu un atac per inundació i un atac amb servidor DHCP fals. Quin dany pot causar cada un?
8. Quina diferència hi ha entre una **exclusió** i una **reserva**?
9. Pot un mateix equip ser client i servidor DHCP alhora? Posa un exemple.
10. Per què el servidor DHCP ha de tenir una IP estàtica?
11. Quina comanda permet comprovar que la sintaxi del `dhcpd.conf` és correcta?
12. Compara el client DHCP de Windows i el de GNU/Linux.
