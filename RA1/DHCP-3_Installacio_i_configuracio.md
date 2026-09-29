# Serveis de Xarxa · RA1 · DHCP (Part 3)

## Instal·lació i configuració d'un servei DHCP

**Cicle:** 2n SMX (Sistemes Microinformàtics i Xarxes)
**Mòdul:** Serveis de Xarxa (M01 · UF 1)
**Resultat d'aprenentatge 1:** Instal·lació i configuració d'un servei DHCP

En aquesta unitat aprendrem a **instal·lar i configurar un servidor DHCP**, un dels serveis de xarxa fonamentals per a la gestió dinàmica d'adreces IP.

---

## Índex

1. [Instal·lació del servei DHCP](#1-installació-del-servei-dhcp)
2. [Modes de funcionament: stand alone vs. xinetd](#2-modes-de-funcionament-stand-alone-vs-xinetd)
3. [Gestió de l'estat d'un servei](#3-gestió-de-lestat-dun-servei)
4. [Tasques principals de configuració](#4-tasques-principals-de-configuració)
5. [Configuració bàsica](#5-configuració-bàsica)
6. [Configuració avançada](#6-configuració-avançada)
7. [Àmbits de configuració](#7-àmbits-de-configuració)
8. [Precedència de les opcions DHCP](#8-precedència-de-les-opcions-dhcp)
9. [Resum i comandes](#9-resum-i-comandes)
10. [Preguntes de repàs](#10-preguntes-de-repàs)

---

## 1. Instal·lació del servei DHCP

El servei DHCP s'estructura en forma de **client/servidor**:

- El **programari client** normalment ja **ve integrat al sistema operatiu**, com a part del servei de xarxa.
- Instal·lar un servei DHCP vol dir **instal·lar i configurar específicament el programari del servidor**.

### 1.1 Instal·lació del servidor

```bash
# Debian / Ubuntu
apt install isc-dhcp-server

# RHEL / Fedora
dnf install dhcp-server

# Habilitar el servei a l'inici
systemctl enable isc-dhcp-server
```

### 1.2 Aplicacions servidor DHCP

Abans de posar en marxa un nou servei de xarxa, cal **analitzar les aplicacions disponibles al mercat**. L'administrador n'ha d'estudiar les característiques:

| Criteri | Què s'avalua |
|---|---|
| **Eficiència** | Rendiment i consum de recursos del sistema. |
| **Cost** | Llicències, suport i manteniment associats. |
| **Reputació** | Opinions i experiències d'altres administradors. |

Normalment, l'administrador acaba utilitzant l'aplicació que li proporciona **el propi sistema operatiu**.

> **Per ampliar:** `isc-dhcp-server` és el servidor clàssic que veurem a classe. L'ISC (l'organització que el desenvolupa) en va aturar el manteniment actiu i recomana el seu successor, **Kea DHCP**. Tot i això, `isc-dhcp-server` continua present a moltes distribucions i és molt útil per aprendre els conceptes.

---

## 2. Modes de funcionament: stand alone vs. xinetd

Un servei de xarxa pot funcionar de dues maneres:

| | **Stand alone** | **Superservei (inetd / xinetd)** |
|---|---|---|
| **Com funciona** | El servidor **escolta directament** les connexions i les atén ell mateix. Cada servei s'executa de manera **independent i contínua**. | El **superservei** detecta les connexions entrants i **activa el dimoni** del servei. Un cop atesa la petició, el dimoni **acaba** i el superservei torna a escoltar. |
| **Processos actius** | Un per cada servei. | Només un (el xinetd). |

> La majoria de serveis suporten **tots dos modes**. És decisió de l'administrador triar el més adequat.

```bash
# Veure processos stand alone actius
ps aux | grep dhcpd

# Estat d'un servei stand alone
systemctl status isc-dhcp-server

# Estat del superservei
systemctl status xinetd

# Exemple de fitxer de configuració xinetd
cat /etc/xinetd.d/tftp
```

### 2.1 Per què el mode stand alone és menys eficient?

- Si volem oferir **n serveis de xarxa** en mode stand alone, cal tenir **engegats els n servidors** alhora.
- Tots ocupen **memòria i CPU**, encara que no estiguin atenent cap client: estan engegats "per si de cas".
- Tenir molts serveis stand alone engegats és **poc eficient** en recursos.

```bash
# Veure tots els serveis actius
ps aux | grep -E "dhcpd|ftpd|httpd"

# Monitorar CPU/RAM del dhcpd
top -p $(pgrep dhcpd)

# Veure la memòria disponible
free -h
```

### 2.2 Avantatges del mode xinetd

- **Un sol procés actiu:** el xinetd és l'únic que escolta constantment. El servei només s'executa quan arriba una petició.
- **Fitxers de configuració propis:** els serveis en mode xinetd tenen fitxers de configuració lligats al xinetd.
- **Exemples típics:** funcionen dins xinetd **telnet** i **tftp**; funcionen individualment (stand alone) **httpd** i **ssh**.

> **Nota:** el servidor DHCP funciona normalment en mode **stand alone**, perquè ha d'estar sempre escoltant al port 67 i respondre ràpidament.

---

## 3. Gestió de l'estat d'un servei

La gestió de l'estat d'un servei inclou normalment aquestes accions:

| Acció | Descripció |
|---|---|
| `start` | Engega el servei. |
| `stop` | Atura el servei. |
| `status` | Mostra l'estat actual. |
| `restart` | Atura i torna a engegar el servei (reinicia). |
| `reload` | Recarrega la configuració **sense aturar** el servei. |
| `enable` | Fa que el servei s'iniciï **automàticament en arrencar el sistema**. |

```bash
systemctl start isc-dhcp-server
systemctl stop isc-dhcp-server
systemctl status isc-dhcp-server
systemctl restart isc-dhcp-server
systemctl reload isc-dhcp-server
systemctl enable isc-dhcp-server      # per iniciar automàticament
```

> **Important:** que un servei estigui **engegat** no garanteix que s'iniciï **automàticament en arrencar** el sistema. Cal configurar-ho (amb `enable`, o els nivells d'execució en sistemes antics).

---

## 4. Tasques principals de configuració

| Tasca | Descripció |
|---|---|
| **Instal·lar** | Instal·lar el programari del servidor DHCP. |
| **Activar / Desactivar** | Gestionar l'estat del servei. |
| **Observar i modificar** | Consultar i canviar la configuració actual. |
| **Monitorar** | Revisar els logs i els fitxers de concessions (*leases*). |

```bash
apt install isc-dhcp-server        # instal·lar
systemctl start isc-dhcp-server    # activar
systemctl stop isc-dhcp-server     # desactivar
nano /etc/dhcp/dhcpd.conf          # observar i modificar configuració
cat /var/lib/dhcp/dhcpd.leases     # monitorar concessions
journalctl -u isc-dhcp-server      # revisar logs
```

---

## 5. Configuració bàsica

### 5.1 Elements principals del `dhcpd.conf`

| Element | Descripció |
|---|---|
| **Opcions globals** | Actualitzacions dinàmiques dels clients i el DNS per a **tot el servidor**. |
| **Definició de subxarxa** | Blocs de subxarxa i subxarxes que atén el servidor. |
| **Opcions genèriques de subxarxa** | Router, màscara de xarxa i domini per als equips de cada subxarxa. |
| **Opcions DHCP** | Interval d'adreces IP dinàmiques i temps màxim de concessió. |

> Perquè el servidor pugui arrencar, ha de saber **a quina xarxa donarà servei** i **quin interval d'adreces** pot usar dinàmicament.

### 5.2 Exemple de `dhcpd.conf` bàsic

```conf
# /etc/dhcp/dhcpd.conf

# --- Opcions globals ---
ddns-update-style none;
default-lease-time 600;
max-lease-time 7200;

# --- Definició de subxarxa ---
subnet 192.168.1.0 netmask 255.255.255.0 {
  # Opcions genèriques
  option routers 192.168.1.1;
  option subnet-mask 255.255.255.0;
  option domain-name "exemple.local";

  # Interval dinàmic
  range 192.168.1.100 192.168.1.200;
}
```

> **Compte amb la sintaxi:** cada declaració acaba amb **punt i coma (`;`)** i els blocs van entre **claus `{ }`**. Oblidar un `;` és l'error més comú.

### 5.3 Entrades individualitzades per a hosts

Els hosts s'identifiquen de manera única per la seva **adreça MAC**, i això permet:

- Mantenir una **IP fixa** per a un equip concret.
- Aplicar **opcions individualitzades** a cada host.

> Les **opcions individuals prevalen** sobre les genèriques de la subxarxa.

```conf
host pc-comptabilitat {
  hardware ethernet 00:1A:2B:3C:4D:5E;   # adreça MAC
  fixed-address 192.168.1.50;             # IP fixa reservada
  option host-name "pc-comptabilitat";    # opció individual
}
```

Per obtenir l'adreça MAC d'una interfície:

```bash
ip link show eth0
```

---

## 6. Configuració avançada

El protocol DHCP permet configuracions força complexes. Les principals característiques avançades són:

### 6.1 Grups i classes

Permeten **agrupar entrades** en grups i classes per aplicar **opcions comunes** a conjunts de clients.

```conf
group {
  option domain-name "dept.local";
  host pc1 { hardware ethernet AA:BB:CC:DD:EE:01; fixed-address 192.168.1.10; }
  host pc2 { hardware ethernet AA:BB:CC:DD:EE:02; fixed-address 192.168.1.11; }
}
```

Tots els hosts del grup hereten l'opció `domain-name "dept.local"` sense haver de repetir-la.

### 6.2 Actualitzacions DDNS (DNS dinàmic)

El DHCP es pot **comunicar amb el servidor DNS** per **crear entrades DNS automàticament** quan un equip rep una configuració DHCP. Així els equips es poden localitzar pel nom.

```conf
ddns-update-style interim;
ddns-domainname "exemple.local.";
ddns-rev-domainname "in-addr.arpa.";
```

---

## 7. Àmbits de configuració

Els clients es poden agrupar en diversos **àmbits** per definir les opcions que han de rebre. El servidor DHCP pot actuar de manera diferent segons l'àmbit. Els àmbits principals són:

1. Subxarxes
2. Període de concessió
3. Adreces reservades
4. PXE
5. Opcions generals

### 7.1 Subxarxes (subnets) i DHCP Relay

El servei DHCP permet l'assignació dinàmica d'adreces per a **diferents subxarxes sense un servidor específic per a cadascuna**.

- Pot donar servei a xarxes **"llunyanes"** que han de **creuar almenys un router**: és el concepte de **DHCP Relaying**.
- Per cada subxarxa cal conèixer l'**adreça de xarxa** i la **màscara**.
- El servidor pot tenir **un o més intervals** dinàmics per subxarxa.

**Per què cal un relay?** El Discover és un broadcast, i **els routers no reenvien broadcasts**. Un **agent de relay** (normalment el router) rep el broadcast del client i el reenvia com a unicast al servidor DHCP.

```text
  [Client]──broadcast──▶[Router amb relay]──unicast──▶[Servidor DHCP]
  10.0.0.x                 (ip helper-address)          192.168.1.254
```

```conf
# Subxarxa local
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
}

# Subxarxa remota (via DHCP Relay)
subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.50 10.0.0.150;
  option routers 10.0.0.1;
}
```

Configuració al router (equipament Cisco) per activar el relay agent:

```text
ip helper-address 192.168.1.254
```

### 7.2 Període de concessió

| Paràmetre | Significat |
|---|---|
| `default-lease-time` | Temps de concessió **per defecte**, quan el client **no ha sol·licitat** cap període concret. |
| `max-lease-time` | Temps de concessió **màxim permès**. Cap client el pot superar. |

- Les concessions poden anar **des de zero segons fins a temps infinit**.
- Un cop exhaurit el temps, cal **renegociar** la concessió.
- Es poden definir **globalment** o **per subxarxa**, segons els requisits.

```conf
# Global
default-lease-time 3600;    # 1 hora per defecte
max-lease-time 86400;       # màxim 24 hores

subnet 192.168.1.0 netmask 255.255.255.0 {
  default-lease-time 600;   # 10 min per a aquesta subxarxa
  max-lease-time 1800;      # màxim 30 min
  range 192.168.1.100 192.168.1.200;
}
```

```bash
# Veure concessions actives
cat /var/lib/dhcp/dhcpd.leases
```

### 7.3 Adreces reservades

A part de l'assignació dinàmica, el servidor pot fer **assignacions dinàmiques fixes**: assignar **sempre la mateixa IP** a un equip concret. Això s'anomena **adreça reservada**.

- **Identificació per MAC:** cal identificar el host de manera **única i inequívoca**.
- **Consum d'adreça:** l'equip "consumeix" l'adreça reservada **tant si està engegat com si està apagat**.

```conf
host impressora-oficina {
  hardware ethernet 00:11:22:33:44:55;
  fixed-address 192.168.1.20;
}

host servidor-web {
  hardware ethernet AA:BB:CC:11:22:33;
  fixed-address 192.168.1.5;
}
```

```bash
# Llistar interfícies i MAC
ip link show

# Taula ARP: IP a MAC
arp -n
```

### 7.4 PXE: protocol d'arrencada via xarxa

**PXE** (*Preboot eXecution Environment*) és un protocol molt utilitzat per arrencar **clients de xarxa "tontos"**: equips que arrenquen **sense sistema operatiu** i el carreguen per la xarxa.

**Com funciona:**

1. **Client → DHCP:** el client demana configuració i el servidor li dona també el **nom d'un fitxer** (opció `filename`) i l'adreça del servidor TFTP (`next-server`).
2. **Servidor → fitxer:** el fitxer indicat acostuma a ser el sistema operatiu o el programari d'inicialització.
3. **Descàrrega TFTP:** el client **descarrega el fitxer via TFTP**.

```conf
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;

  # Opcions PXE
  next-server 192.168.1.254;        # IP del servidor TFTP
  filename "pxelinux.0";            # fitxer d'arrencada
}
```

```bash
# Instal·lar el servidor TFTP
apt install tftpd-hpa
systemctl start tftpd-hpa
ls /var/lib/tftpboot/               # directori dels fitxers PXE
```

### 7.5 Opcions generals (options)

El DHCP no només dona IP, màscara i gateway: subministra molts altres paràmetres en forma d'**options**.

| Opció | Descripció |
|---|---|
| `routers` | Porta d'enllaç predeterminada. |
| `subnet-mask` | Màscara de subxarxa. |
| `domain-name-servers` | Servidors DNS (servidors de noms de domini). |
| `domain-name` | Nom de domini. |
| `broadcast-address` | Adreça de broadcast. |
| `ntp-servers` | Servidors de sincronització horària (opció específica). |
| **Específiques** | Paràmetres molt concrets per a casos particulars. |

```conf
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option subnet-mask 255.255.255.0;             # màscara
  option routers 192.168.1.1;                   # gateway
  option domain-name-servers 8.8.8.8, 8.8.4.4;  # DNS
  option domain-name "empresa.local";           # domini
  option broadcast-address 192.168.1.255;       # broadcast
  option ntp-servers 192.168.1.1;               # NTP (específica)
}
```

---

## 8. Precedència de les opcions DHCP

Les opcions es poden definir a tres nivells. Si una mateixa opció apareix en més d'un, **guanya la més concreta**:

| Nivell | Abast | Prioritat |
|---|---|---|
| **Àmbit global** | Opcions per a **totes** les subxarxes. | Més baixa |
| **Àmbit de subxarxa** | Opcions específiques d'**una subxarxa**. | Mitjana |
| **Àmbit d'host** | Opcions individuals per a **un sol equip**. | **Més alta** |

```text
   HOST  >  SUBXARXA  >  GLOBAL
  (més concret)        (més general)
```

**Exemple pràctic:**

```conf
option domain-name "empresa.local";          # global

subnet 192.168.1.0 netmask 255.255.255.0 {
  option domain-name "oficina.empresa.local";   # sobreescriu el global
  range 192.168.1.100 192.168.1.200;

  host pc-direccio {
    hardware ethernet AA:BB:CC:00:00:01;
    fixed-address 192.168.1.10;
    option domain-name "direccio.empresa.local"; # sobreescriu el de subxarxa
  }
}
```

Un client normal de la subxarxa rebrà `oficina.empresa.local`; el `pc-direccio` rebrà `direccio.empresa.local`.

---

## 9. Resum i comandes

| Concepte | Idea clau |
|---|---|
| **Arquitectura** | Model client/servidor. Client integrat al SO; el servidor requereix instal·lació i configuració específica. |
| **Modes** | Stand alone (menys eficient) o xinetd (superservei, més eficient en recursos). |
| **Configuració** | Subxarxes, períodes de concessió, adreces reservades per MAC, PXE i opcions per àmbit. |

### Resum de comandes Linux

```bash
# Instal·lació
apt install isc-dhcp-server

# Gestió del servei
systemctl start|stop|restart|reload|status isc-dhcp-server
systemctl enable isc-dhcp-server

# Fitxers clau
nano /etc/dhcp/dhcpd.conf          # configuració principal
cat /var/lib/dhcp/dhcpd.leases     # registre de concessions

# Diagnosi
journalctl -u isc-dhcp-server      # logs del servei
ip link show                       # veure MAC
arp -n                             # taula ARP
dhcpd -t -cf /etc/dhcp/dhcpd.conf  # comprovar la sintaxi
```

### Fitxers importants

| Fitxer | Funció |
|---|---|
| `/etc/dhcp/dhcpd.conf` | Configuració principal del servidor. |
| `/etc/default/isc-dhcp-server` | Interfície(s) per les quals escolta el servidor. |
| `/var/lib/dhcp/dhcpd.leases` | Registre de concessions del servidor. |
| `/var/lib/dhcp/dhclient.leases` | Registre de concessions del client. |

---

## 10. Preguntes de repàs

1. Per què no cal instal·lar el programari client DHCP, però sí el del servidor?
2. Quins criteris cal valorar abans d'escollir una aplicació servidor DHCP?
3. Explica la diferència entre el mode **stand alone** i el mode **superservei (xinetd)**. Quin és més eficient en recursos i per què?
4. Quina diferència hi ha entre `restart` i `reload`? I entre `start` i `enable`?
5. Quins són els quatre elements principals del fitxer `dhcpd.conf`?
6. Escriu la configuració d'una subxarxa `192.168.10.0/24` amb un interval del `.50` al `.150`, porta d'enllaç `.1` i DNS `8.8.8.8`.
7. Escriu una reserva per a una impressora amb MAC `00:11:22:33:44:55` i IP `192.168.10.20`.
8. Què fan `default-lease-time` i `max-lease-time`? Quina diferència hi ha?
9. Què és un **DHCP Relay** i per què és necessari quan hi ha subxarxes remotes?
10. Explica el procés d'arrencada **PXE**. Quines opcions del `dhcpd.conf` intervenen?
11. Què és el **DDNS** i quin avantatge aporta?
12. Ordena per prioritat (de més a menys) els àmbits global, subxarxa i host. Posa un exemple.
