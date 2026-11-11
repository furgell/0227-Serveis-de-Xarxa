&#91;image: Generalitat de Catalunya · Institut Jaume Huguet, Valls\]

# Pràctica 2: Servidor DNS a Ubuntu Server

Nov 4, 2026 · @Ferran

CFGM Sistemes Microinformàtics i Xarxes (SMX) · Mòdul 7: Serveis de xarxa · A2: Servei de resolució de noms · Institut Jaume Huguet (Valls) · 2n SMX

## Objectius i entorn

Instal·lareu i configurareu un servidor DNS amb **Bind9** a **Ubuntu Server 24.04** i en comprovareu el funcionament des d'un client **Ubuntu Desktop 22.04**. Tot es fa amb escriptoris virtuals d'IsardVDI.

Al final de la pràctica haureu de saber:

- Configurar una IP estàtica a la interfície correcta del servidor amb netplan.
- Instal·lar el servei DNS (Bind9).
- Crear una zona de cerca directa i una d'inversa.
- Crear registres A, PTR i CNAME.
- Configurar un reenviador.
- Comprovar la resolució amb `ping`, `nslookup` i `dig`.

### Entorn a IsardVDI

Cada escriptori té dues interfícies de xarxa. Comproveu-ne els noms amb `ip a`:

- **Interfície 1 (`enp1s0`)**: xarxa per defecte d'Isard. Rep la IP per DHCP i dóna sortida a Internet. No la toqueu.
- **Interfície 2 (`enp2s0`)**: xarxa interna que comparteixen el servidor i el client. És la que configurareu.

La configuració de xarxa de les plantilles d'Isard és al fitxer **`/etc/netplan/50-cloud-init.yaml`**.

### Pla d'adreces

**X** = el vostre número de llista. Tots els valors d'aquesta pràctica fan servir **X=2** com a exemple. Als fitxers i ordres de la pràctica la IP surt com a 192.168.X: substituïu sempre la X pel vostre número (amb X=2, 192.168.X.1 és 192.168.2.1, la zona inversa és 2.168.192.in-addr.arpa i el fitxer, db.192.168.2). Si la deixeu com a X, Bind9 i netplan donaran error.

| Equip | Nom (hostname) | Interfície 2 | Màscara | Porta d'enllaç | DNS |
| --- | --- | --- | --- | --- | --- |
| Servidor (Ubuntu Server 24.04) | servercognom | 192.168.X.1 | /24 (255.255.255.0) | (buida) | 192.168.X.1 |
| Client 1 (Ubuntu Desktop 22.04) | pc01 | 192.168.X.10 | /24 (255.255.255.0) | (buida) | 192.168.X.1 |
| Client 2 (Ubuntu Desktop 22.04, opcional) | pc02 | 192.168.X.11 | /24 (255.255.255.0) | (buida) | 192.168.X.1 |

La porta d'enllaç de la interfície 2 es deixa buida. La sortida a Internet ja la dóna la interfície 1, i dues portes d'enllaç fan que la xarxa falli.

El servidor surt a Internet per la interfície 1 i dóna servei DNS als clients per la interfície 2. El domini de la pràctica és **smx.com**.

## Part 1: Preparar el servidor

Totes les ordres del servidor es fan com a root. Entreu-hi amb `sudo -i`.

### 1.1 Canviar el nom del servidor

1. Posa-li el nom **servercognom**, on cognom és el vostre cognom (en minúscules, sense accents ni espais). Exemple: `servergarcia`.

   ```bash
   hostnamectl set-hostname servergarcia
   ```
2. Edita `/etc/hosts` i canvia el nom antic de la línia `127.0.1.1` pel nou:

   ```bash
   nano /etc/hosts
   ```

   ```
   127.0.0.1   localhost
   127.0.1.1   servergarcia
   ```
3. Comprova-ho amb `hostname` i reinicia amb `reboot`.

### 1.2 IP estàtica a la interfície 2

1. Mira el nom de les interfícies amb `ip a`. La interfície 2 (`enp2s0`) sovint surt sense IP (`DOWN` o sense `inet`).
2. Fes una còpia de seguretat del fitxer de netplan i obre'l:

   ```bash
   cp /etc/netplan/50-cloud-init.yaml /root/50-cloud-init.yaml.bak
   nano /etc/netplan/50-cloud-init.yaml
   ```
3. Deixa la interfície 1 com està (DHCP) i afegeix-hi la interfície 2 amb les dades del pla d'adreces:

   ```yaml
   network:
     version: 2
     ethernets:
       enp1s0:
         dhcp4: true
       enp2s0:
         dhcp4: false
         addresses:
           - 192.168.X.1/24
         nameservers:
           addresses: [192.168.X.1]
           search: [smx.com]
   ```
   - **addresses**: 192.168.X.1/24 (el /24 és la màscara 255.255.255.0).
   - **Porta d'enllaç**: no n'hi poseu (no hi ha cap `routes` ni `gateway4` a la interfície 2).
   - **nameservers**: 192.168.X.1, la IP del mateix servidor. No hi poseu 127.0.0.1: Ubuntu envia les consultes DNS per la interfície on està configurat el DNS (enp2s0), i el 127.0.0.1 només existeix a la interfície de loopback (lo), així que no funcionaria.
   - Al YAML només es fan servir **espais**, mai tabulacions, i el sagnat ha de quadrar exactament.
4. Aquest fitxer el genera **cloud-init** i es pot sobreescriure en reiniciar. Perquè no es perdin els canvis, desactiva la configuració de xarxa de cloud-init:

   Crea un fitxer nou amb nano:

   ```bash
   nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
   ```

   Escriu-hi aquesta única línia, desa (Ctrl+O, Intro) i surt (Ctrl+X):

   ```
   network: {config: disabled}
   ```
5. Aplica els canvis:

   ```bash
   chmod 600 /etc/netplan/50-cloud-init.yaml
   netplan apply
   ```
6. Comprova-ho amb `ip a`. La interfície 2 ha de mostrar `inet 192.168.X.1/24`, i la 1 ha de seguir tenint IP i Internet (`ping -c 3 8.8.8.8`).

### 1.3 Tallafoc

A Ubuntu el tallafoc (**ufw**) ve desactivat per defecte i el ping funciona. No el desactiveu si està actiu: obriu només el que cal.

1. Comprova l'estat:

   ```bash
   ufw status
   ```
2. Si surt `Status: active`, obre el port del DNS (53) i el de SSH:

   ```bash
   ufw allow 53
   ufw allow ssh
   ```

## Part 2: Instal·lar el servei DNS (Bind9)

1. Actualitza la llista de paquets i instal·la Bind9 i les eines de comprovació:

   ```bash
   apt update
   apt install bind9 bind9utils dnsutils -y
   ```
2. Comprova que el servei està en marxa:

   ```bash
   systemctl status bind9
   ```

   Ha de sortir **active (running)**. Surt amb `q`.
3. Mira els fitxers de configuració que s'han creat:

   ```bash
   ls /etc/bind/
   ```

> Si falla la instal·lació, comprova que la interfície 1 té Internet amb `ping -c 3 www.google.com`.

### 2.1 Fer que el DNS escolti només a la xarxa interna

1. Obre el fitxer d'opcions:

   ```bash
   nano /etc/bind/named.conf.options
   ```
2. Dins del bloc `options { ... };`, abans del `};` final, afegeix qui pot fer consultes i a quines IP escolta el servidor:

   ```
   options {
           directory "/var/cache/bind";
   
           dnssec-validation auto;
   
           listen-on { 127.0.0.1; 192.168.X.1; };
           listen-on-v6 { none; };
           allow-query { localhost; 192.168.X.0/24; };
   };
   ```
   - **listen-on**: només escolta a la IP de la interfície 2 (i a ell mateix), no a la xarxa d'Isard.
   - **allow-query**: només respon als equips de la xarxa 192.168.X.0/24.
3. Comprova la sintaxi i reinicia el servei:

   ```bash
   named-checkconf
   systemctl restart bind9
   ```

   Si `named-checkconf` no diu res, la sintaxi és correcta. Si surt un error, indica el fitxer i la línia (sovint falta un `;`).
4. Comprova que escolta al port 53 de la interfície 2:

   ```bash
   ss -tulpn | grep :53
   ```

   Hi ha de sortir `192.168.X.1:53` amb el procés `named`.

## Part 3: Zones de cerca directa i inversa

Una zona és la part de l'espai de noms DNS que gestiona el servidor.

- **Zona de cerca directa**: tradueix noms a IP (`smxservidor.smx.com` → `192.168.X.1`).
- **Zona de cerca inversa**: tradueix IP a noms (`192.168.X.1` → `smxservidor.smx.com`).

A Bind9 cada zona es fa en dos passos: es **declara** a `named.conf.local` i es crea el seu **fitxer de zona** a `/etc/bind/`.

### 3.1 Declarar les zones

1. Obre el fitxer de zones locals:

   ```bash
   nano /etc/bind/named.conf.local
   ```
2. Afegeix-hi les dues zones:

   ```
   zone "smx.com" {
           type master;
           file "/etc/bind/db.smx.com";
   };
   
   zone "X.168.192.in-addr.arpa" {
           type master;
           file "/etc/bind/db.192.168.X";
   };
   ```
   - **type master**: és una zona principal (l'equivalent de *Zona principal* a Windows).
   - El nom de la zona inversa s'escriu amb els tres primers octets **al revés** seguit de `.in-addr.arpa`: per a 192.168.X queda `X.168.192.in-addr.arpa`.
3. Comprova la sintaxi amb `named-checkconf`.

### 3.2 Zona directa smx.com

1. Parteix de la plantilla `db.local`:

   ```bash
   cp /etc/bind/db.local /etc/bind/db.smx.com
   nano /etc/bind/db.smx.com
   ```
2. Deixa-la així (separa les columnes amb tabulacions o espais; a Bind9 funcionen igual):

   ```
   $TTL    604800
   @       IN      SOA     smxservidor.smx.com. root.smx.com. (
                                 1         ; Serial
                            604800         ; Refresh
                             86400         ; Retry
                           2419200         ; Expire
                            604800 )       ; Negative Cache TTL
   ;
   @       IN      NS      smxservidor.smx.com.
   ```
   - **SOA**: indica el servidor principal de la zona i el correu de l'administrador (`root.smx.com.` vol dir root@smx.com).
   - **NS**: el servidor de noms de la zona.
   - Fixeu-vos en el **punt final** dels noms complets. Sense punt, Bind9 hi afegeix el domini al darrere.

### 3.3 Zona inversa 192.168.X

1. Parteix de la plantilla `db.127`:

   ```bash
   cp /etc/bind/db.127 /etc/bind/db.192.168.X
   nano /etc/bind/db.192.168.X
   ```
2. Deixa-la així:

   ```
   $TTL    604800
   @       IN      SOA     smxservidor.smx.com. root.smx.com. (
                                 1         ; Serial
                            604800         ; Refresh
                             86400         ; Retry
                           2419200         ; Expire
                            604800 )       ; Negative Cache TTL
   ;
   @       IN      NS      smxservidor.smx.com.
   ```

Encara no reinicieu el servei: el registre NS apunta a `smxservidor`, que es crea a la Part 4.

## Part 4: Registre del servidor i comprovació

### 4.1 Host smxservidor

1. Afegeix al final de `/etc/bind/db.smx.com` el registre **A** del servidor:

   ```
   smxservidor     IN      A       192.168.X.1
   ```

   El FQDN queda `smxservidor.smx.com`, perquè el nom no porta punt final i Bind9 hi afegeix el domini.
2. Afegeix al final de `/etc/bind/db.192.168.X` el registre **PTR** associat:

   ```
   1       IN      PTR     smxservidor.smx.com.
   ```

   L'`1` és l'últim octet de la IP del servidor (192.168.X.**1**). A Windows això ho fa la casella *Crear registro del puntero (PTR) asociado*; a Bind9 s'escriu a mà.
3. Comprova les dues zones:

   ```bash
   named-checkzone smx.com /etc/bind/db.smx.com
   named-checkzone X.168.192.in-addr.arpa /etc/bind/db.192.168.X
   ```

   Totes dues han de dir `OK`.
4. Reinicia el servei i comprova'n l'estat:

   ```bash
   systemctl restart bind9
   systemctl status bind9
   ```

   Si surt un error, mira'l amb `journalctl -u named -e`.

> **Important:** cada cop que modifiqueu un fitxer de zona, **augmenteu el Serial** (1 → 2 → 3...) i reinicieu el servei.

### 4.2 Comprovació des del servidor

Executa al servidor:

```bash
nslookup smxservidor.smx.com 127.0.0.1
nslookup 192.168.X.1 127.0.0.1
dig smxservidor.smx.com
```

Resultat esperat:

- La primera ordre retorna `Address: 192.168.X.1`.
- La segona retorna `name = smxservidor.smx.com.`
- A `dig` surt `status: NOERROR` i una `ANSWER SECTION` amb la IP. Si surt `NXDOMAIN`, revisa el nom del registre i el punt final; si la inversa no respon, falta la zona inversa o el registre PTR.

## Part 5: Client Ubuntu Desktop i comprovació

El client ha d'estar a la mateixa xarxa interna d'Isard que el servidor (interfície 2). Obriu un terminal (Ctrl+Alt+T).

### 5.1 Configurar la xarxa del client

1. Canvia el nom del client a **pc01** i comprova les interfícies:

   ```bash
   sudo hostnamectl set-hostname pc01
   ip a
   ```
2. Edita el fitxer de netplan:

   ```bash
   sudo nano /etc/netplan/50-cloud-init.yaml
   ```

   ```yaml
   network:
     version: 2
     ethernets:
       enp1s0:
         dhcp4: true
       enp2s0:
         dhcp4: false
         addresses:
           - 192.168.X.10/24
         nameservers:
           addresses: [192.168.X.1]
           search: [smx.com]
   ```
   - **addresses**: 192.168.X.10/24.
   - **Porta d'enllaç**: buida.
   - **nameservers**: 192.168.X.1, la IP del servidor.
3. Desactiva cloud-init per a la xarxa i aplica els canvis:

   ```bash
   sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
   ```

   Escriu-hi la línia `network: {config: disabled}`, desa i surt. Després aplica els canvis:

   ```bash
   sudo chmod 600 /etc/netplan/50-cloud-init.yaml
   sudo netplan apply
   ```
4. Desactiva la interfície 1 del client mentre facis les proves. Si no, el client pot preguntar al DNS d'Isard en lloc del vostre i les proves enganyen.

   ```bash
   sudo ip link set enp1s0 down
   ```

   En reiniciar el client es torna a activar.

### 5.2 Comprovacions des del client

A Ubuntu, `/etc/resolv.conf` sempre mostra `127.0.0.53` (el gestiona systemd-resolved). El DNS real es mira amb `resolvectl status`.

Executa al client:

```bash
ip a
resolvectl status
ping -c 4 192.168.X.1
ping -c 4 smxservidor.smx.com
nslookup smxservidor.smx.com
nslookup 192.168.X.1
```

| Ordre | Què ha de sortir |
| --- | --- |
| `ip a` | IP 192.168.X.10/24 a `enp2s0` |
| `resolvectl status` | `DNS Servers: 192.168.X.1` i `DNS Domain: smx.com` a `enp2s0` |
| `ping -c 4 192.168.X.1` | 4 respostes, 0% perduts |
| `ping -c 4 smxservidor.smx.com` | Fa ping a 192.168.X.1 (el nom es resol amb el vostre DNS) |
| `nslookup smxservidor.smx.com` | `Address: 192.168.X.1` |
| `nslookup 192.168.X.1` | `name = smxservidor.smx.com.` |

Si el ping a la IP falla, revisa la IP i la xarxa de l'escriptori a Isard. Si el ping a la IP funciona però el nom no es resol, revisa el DNS del client i el tallafoc de la Part 1.3.

> Si teniu un segon client (pc02, 192.168.X.11), repetiu la Part 5 amb ell.

## Exercicis

Aquests exercicis no tenen els passos escrits. Investigueu com fer-los i documenteu-los amb captures i explicacions. Recordeu augmentar el Serial i reiniciar Bind9 cada cop que toqueu una zona.

### Exercici 1: Àlies www

Crea un àlies (registre **CNAME**) anomenat `www` que apunti a `smxservidor.smx.com`.

- Explica què és un àlies i per a què serveix.
- Comprova-ho des del client amb `nslookup www.smx.com` i `ping -c 4 www.smx.com`.

### Exercici 2: Host del client pc01

Crea al vostre DNS un host `pc01` que apunti a la IP del client (192.168.X.10), amb el seu registre PTR.

Comprova-ho des del client:

```bash
nslookup pc01.smx.com
nslookup 192.168.X.10
nslookup www.smx.com
```

### Exercici 3: Reenviador

Configura un reenviador perquè, quan el vostre DNS no pugui resoldre un nom, ho pregunti a un DNS públic (per exemple el de Google, 8.8.8.8).

- Explica la diferència entre usar un reenviador i usar els servidors arrel (*root hints*, fitxer `/etc/bind/db.root`).
- Comprova-ho des del client, amb la interfície 1 encara desactivada: `nslookup www.google.com`.

> **Pista:** fitxer `named.conf.options`, bloc `forwarders`. Penseu en els DNS dels proveïdors d'Internet (ISP) i en el de Google. Si teniu `allow-query` i `listen-on` com a la Part 2.1, el client ja hi pot preguntar.

### Exercici 4: Noms sense punt final

Observeu aquest fitxer de zona del domini `webebre.net`:

```
$TTL    604800
@       IN      SOA     webebre.net. root.webebre.net. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@               IN      NS      webebre.net.
webebre.net     IN      A       192.168.83.1
dns.webebre.net IN      A       192.168.83.1
pc1.webebre.net IN      A       192.168.83.1
```

Expliqueu què passa amb les dues últimes línies (`dns.webebre.net` i `pc1.webebre.net`): quin nom complet acaba resolent realment Bind9 i com s'haurien d'escriure correctament. Proveu-ho al vostre servidor amb un registre equivalent a `smx.com` i comproveu-ho amb `nslookup`.

## Comprovació final

En acabar, feu aquestes comprovacions i poseu-les juntes en un apartat del manual. Cada captura porta el seu codi al peu. El professorat corregeix a partir d'aquesta taula.

| Codi | On | Ordre | Què ha de sortir |
| --- | --- | --- | --- |
| E1 | Servidor | `hostname; ip -br a` | `servercognom` i `enp2s0` amb `192.168.X.1/24`; `enp1s0` amb la IP d'Isard |
| E2 | Servidor | `cat /etc/netplan/50-cloud-init.yaml` | `enp2s0` amb `192.168.X.1/24`, sense `routes` ni `gateway4` |
| E3 | Servidor | `ping -c 2 8.8.8.8` | 0% packet loss |
| E4 | Servidor | `systemctl is-active bind9; named-checkconf && echo OK` | `active` i `OK` |
| E5 | Servidor | `ss -tulpn \| grep named` | `127.0.0.1:53` i `192.168.X.1:53`, cap línia amb la IP d'Isard |
| E6 | Servidor | `named-checkzone smx.com /etc/bind/db.smx.com` i el mateix per a la inversa | `loaded serial N` i `OK` a totes dues |
| E7 | Servidor | `nslookup smxservidor.smx.com 127.0.0.1` | `Address: 192.168.X.1` |
| E8 | Servidor | `nslookup 192.168.X.1 127.0.0.1` | `name = smxservidor.smx.com.` |
| E9 | Client | `ip a; resolvectl status` | `192.168.X.10/24` i `DNS Servers: 192.168.X.1` a `enp2s0` |
| E10 | Client | `ping -c 4 smxservidor.smx.com; nslookup 192.168.X.1` | 0% perduts i `name = smxservidor.smx.com.` |
| E11 | Client | `nslookup www.smx.com` | `canonical name = smxservidor.smx.com.` (Exercici 1) |
| E12 | Client | `nslookup pc01.smx.com; nslookup 192.168.X.10` | `192.168.X.10` i `name = pc01.smx.com.` (Exercici 2) |
| E13 | Client | `nslookup www.google.com` amb `enp1s0` desactivada | Una IP pública, servidor `192.168.X.1` (Exercici 3) |
| E14 | Servidor | `nslookup` del registre de l'Exercici 4 | El nom mal resolt i el corregit |

És normal que `ss` també mostri `127.0.0.53` (systemd-resolved) i el port `953` (rndc, control de Bind9).

### Graella d'avaluació

| Bloc | Evidències | Punts |
| --- | --- | --- |
| Xarxa del servidor (Part 1) | E1, E2, E3 | 1 |
| Bind9 i opcions (Part 2) | E4, E5 | 1 |
| Zones directa i inversa (Parts 3 i 4) | E6 + fitxers de zona | 2 |
| Comprovació des del servidor (4.2) | E7, E8 | 1 |
| Client (Part 5) | E9, E10 | 1,5 |
| Exercici 1: àlies www | E11 + explicació | 0,75 |
| Exercici 2: host pc01 | E12 | 0,75 |
| Exercici 3: reenviador | E13 + explicació | 0,75 |
| Exercici 4: noms sense punt final | E14 + explicació | 0,75 |
| Manual (portada, índex, format, ortografia) | — | 0,5 |
| **Total** |  | **10** |

Una evidència que falta o on no es veu el nom del servidor i la IP compta 0 en aquell bloc.

## Lliurament i avaluació

Lliureu un manual en PDF que anireu fent mentre feu la pràctica.

- Captures de pantalla ordenades de cada pas: instal·lació, configuració i comprovacions. Les captures de la comprovació final porten el seu codi (E1, E2…) al peu.
- Cada captura amb una explicació breu del que es veu i per què es fa.
- Contingut de `50-cloud-init.yaml` (servidor i client), `named.conf.options`, `named.conf.local` i els dos fitxers de zona.
- Portada amb nom, curs i data, i índex.
- Pàgines numerades, text justificat i sense faltes d'ortografia.
- A les captures s'ha de veure el nom del vostre servidor (`servercognom` al prompt) i la vostra IP (192.168.X.…).
