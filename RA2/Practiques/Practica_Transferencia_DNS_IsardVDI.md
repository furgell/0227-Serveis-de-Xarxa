&#91;image: Generalitat de Catalunya · Institut Jaume Huguet, Valls\]

# Pràctica: Transferència de zona DNS a Windows Server (IsardVDI)

CFGM Sistemes Microinformàtics i Xarxes (SMX) · Mòdul 7: Serveis de xarxa · RA2: Servei de resolució de noms · Institut Jaume Huguet (Valls) · 2n SMX

Com ja vam veure a les diapositives de teoria, normalment tindrem configurats com a mínim dos servidors DNS per a cada zona: un **principal** (primari) i un **secundari**. El secundari no es configura registre a registre: obté una còpia de la zona del principal mitjançant una **transferència de zona** (AXFR completa o IXFR incremental). Així, si el principal cau, els clients poden continuar resolent noms.

En aquesta pràctica muntareu tot l'escenari a **IsardVDI**: reaprofitareu el Windows Server de la pràctica 1 com a servidor principal i creareu un segon Windows Server, nou i independent, que farà de servidor secundari.

## 1. Objectius

- Crear a IsardVDI un segon escriptori Windows Server independent del de la pràctica 1.
- Connectar els escriptoris dels servidors i del client a una mateixa xarxa interna d'IsardVDI.
- Configurar el servidor de la pràctica 1 com a DNS principal i autoritzar les transferències de zona només cap al secundari.
- Crear una zona secundària i definir un mètode per comprovar que la transferència funciona i que el secundari respon als clients.

## 2. Escenari de treball

Treballareu amb tres escriptoris d'IsardVDI. Als exemples la zona s'anomena `smx.local`: feu servir **la mateixa zona que vau crear a la pràctica 1**. Si a la pràctica 1 vau fer servir altres adreces IP, mantingueu-les i adapteu la resta de la taula.

| Escriptori IsardVDI | Rol | Xarxes | IP xarxa interna | DNS configurat |
|---|---|---|---|---|
| **DNS1** (pràctica 1) | DNS principal | Default + Personal | 192.168.50.10/24 | 127.0.0.1 |
| **DNS2** (nou) | DNS secundari | Default + Personal | 192.168.50.11/24 | 127.0.0.1 i 192.168.50.10 |
| **CLIENT** | Client DNS (Windows 10/11) | Default + Personal | 192.168.50.20/24 | 192.168.50.10 i 192.168.50.11 |

> **Xarxes a IsardVDI.** La xarxa **Default** dona sortida a Internet per NAT i assigna IP per DHCP: la deixarem tal com està. La xarxa **Personal** és una xarxa privada que només comparteixen els escriptoris del vostre usuari i **no té DHCP**, per això hi posarem IP estàtica. És per aquesta xarxa per on es comunicaran els servidors i el client. Si el professor us indica una altra xarxa interna (per exemple, una xarxa del grup), feu servir aquella.

## 3. Part A – Preparar els escriptoris a IsardVDI

### A.1 Revisar l'escriptori de la pràctica 1 (DNS1)

1. Entreu a IsardVDI i, si l'escriptori DNS1 està engegat, **atureu-lo**: el maquinari només es pot modificar amb l'escriptori apagat.
2. Feu clic a **Editar** i, a l'apartat de maquinari/xarxes, comproveu que té les interfícies **Default** i **Personal**. Si no hi és, afegiu la xarxa Personal i deseu.

### A.2 Crear el servidor secundari (DNS2)

1. Feu clic a **Nou escriptori** i escolliu la plantilla de **Windows Server** (la mateixa versió que a la pràctica 1 o la que indiqui el professor).
2. Poseu-li un nom identificatiu, per exemple `SMX-DNS2-cognom`.
3. A les opcions avançades/maquinari, seleccioneu les xarxes **Default** i **Personal**.
4. Creeu l'escriptori i engegueu-lo. Connecteu-vos amb el visor que preferiu (navegador o RDP).

> **Important:** el DNS2 s'ha de crear des de la plantilla de Windows Server, **no** duplicant el vostre DNS1 ni fent una plantilla a partir d'ell. Un clon portaria el mateix nom d'equip, el mateix identificador de sistema (SID) i la zona ja creada com a principal, i la transferència no tindria cap sentit.

### A.3 Preparar el client

Podeu reaprofitar el client Windows de la pràctica 1 o crear-ne un de nou des d'una plantilla de Windows 10/11. Igual que els servidors, ha de tenir les xarxes **Default** i **Personal**.

> 📷 **Captura 1:** llistat d'escriptoris d'IsardVDI on es vegin DNS1, DNS2 i CLIENT, i la pantalla d'edició d'un d'ells amb les dues xarxes assignades.

## 4. Part B – Configuració bàsica de xarxa

### B.1 Identificar les targetes de xarxa

A cada màquina obriu `ncpa.cpl`. Veureu dues targetes: la de la xarxa Default té una IP obtinguda per DHCP i la de la xarxa Personal tindrà una adreça automàtica `169.254.x.x` (perquè no hi ha DHCP). Canvieu-los el nom a **DEFAULT** i **INTERNA** perquè sigui fàcil distingir-les.

### B.2 Adreçament estàtic a la targeta INTERNA

Configureu la targeta INTERNA de cada màquina segons la taula de l'apartat 2, amb màscara 255.255.255.0 i **sense porta d'enllaç** (la sortida a Internet ja la fa la targeta DEFAULT). Si ho preferiu, ho podeu fer amb PowerShell (exemple per al DNS2):

```powershell
Rename-NetAdapter -Name "Ethernet 2" -NewName "INTERNA"
New-NetIPAddress -InterfaceAlias "INTERNA" -IPAddress 192.168.50.11 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias "INTERNA" -ServerAddresses 127.0.0.1,192.168.50.10
```

### B.3 Nom d'equip i connectivitat

1. Al DNS2, canvieu el nom de l'equip a **DNS2** (Administrador del servidor › Servidor local › Nom de l'equip) i reinicieu.
2. Per poder fer `ping` entre màquines, habiliteu al tallafoc de cada equip la regla **Compartir fitxers i impressores (Sol·licitud d'eco - ICMPv4 d'entrada)**.
3. Comproveu la connectivitat: des del DNS2 feu `ping 192.168.50.10` i des del CLIENT feu ping a tots dos servidors.

> 📷 **Captura 2:** configuració IP de la targeta INTERNA del DNS2 (`ipconfig /all`) i els pings correctes entre les tres màquines.

### B.4 Instal·lar el rol de servidor DNS al DNS2

Administrador del servidor › **Afegir rols i característiques** › marqueu **Servidor DNS** i completeu l'assistent. També es pot fer amb:

```powershell
Install-WindowsFeature DNS -IncludeManagementTools
```

> 📷 **Captura 3:** rol de Servidor DNS instal·lat correctament al DNS2.

## 5. Part C – Configurar el servidor principal (DNS1)

1. Comproveu que la targeta INTERNA del DNS1 té la IP `192.168.50.10`.
2. Obriu l'**Administrador de DNS** i, a la zona `smx.local`, creeu un registre **A** `dns2` que apunti a `192.168.50.11`.
3. Feu clic dret a la zona › **Propietats** › pestanya **Servidors de noms**: afegiu `dns2.smx.local` amb la IP `192.168.50.11`. Això crea el registre NS del secundari.
4. Pestanya **Transferències de zona**: marqueu **Permet les transferències de zona** i escolliu **Només als servidors que apareixen a la pestanya Servidors de noms**.
5. Al mateix lloc, botó **Notificar…**: marqueu **Notifica automàticament** › **Servidors que apareixen a la pestanya Servidors de noms**. Així el principal avisarà el secundari (NOTIFY) cada cop que la zona canviï.
6. Apliqueu els canvis.

Equivalent amb PowerShell:

```powershell
Add-DnsServerResourceRecordA -ZoneName "smx.local" -Name "dns2" -IPv4Address 192.168.50.11
Add-DnsServerResourceRecord -ZoneName "smx.local" -NS -Name "@" -NameServer "dns2.smx.local"
Set-DnsServerPrimaryZone -Name "smx.local" -SecureSecondaries TransferToZoneNameServer -Notify Notify
```

> 📷 **Captura 4:** pestanyes **Servidors de noms** i **Transferències de zona** de les propietats de la zona al DNS1.

> **Ampliació (opcional):** si a la pràctica 1 vau crear una zona de cerca inversa, repetiu el mateix procés per a aquesta zona.

## 6. Part D – Crear la zona secundària (DNS2)

1. Al DNS2, obriu l'**Administrador de DNS** › **Zones de cerca directa** › clic dret › **Zona nova…**
2. Tipus de zona: **Zona secundària**.
3. Nom de la zona: `smx.local` (exactament el mateix nom que al principal).
4. Servidors DNS principals: escriviu `192.168.50.10`. L'assistent l'ha de validar amb un tic verd. Si dona error, reviseu la connectivitat i la configuració de la Part C.
5. Finalitzeu l'assistent.
6. Si la zona apareix amb el missatge que no s'ha carregat, feu clic dret › **Transfereix des del servidor principal** i actualitzeu (F5).

Equivalent amb PowerShell:

```powershell
Add-DnsServerSecondaryZone -Name "smx.local" -ZoneFile "smx.local.dns" -MasterServers 192.168.50.10
Start-DnsServerZoneTransfer -Name "smx.local" -FullTransfer
```

Comproveu que apareixen tots els registres de la zona i que **no els podeu modificar**: una zona secundària és només de lectura.

> 📷 **Captura 5:** assistent de zona nova amb el servidor principal validat i la zona secundària amb tots els registres copiats.

## 7. Part E – Mètode de comprovació

Aquesta és la part més important de la pràctica. Heu de demostrar, amb proves i captures, que **(1)** la transferència de zona funciona i **(2)** el secundari respon correctament als clients. Seguiu aquest procediment:

### E.1 La zona és idèntica als dos servidors

Compareu el número de sèrie del registre SOA consultant cada servidor. Si els dos números coincideixen, el secundari té la mateixa versió de la zona que el principal:

```
nslookup -type=SOA smx.local 192.168.50.10
nslookup -type=SOA smx.local 192.168.50.11
```

> 📷 **Captura 6:** les dues consultes amb el mateix número de sèrie (serial).

### E.2 Els canvis es propaguen

1. Al DNS1, creeu un registre A nou, per exemple `prova` → `192.168.50.99`.
2. Torneu a consultar el SOA al DNS1: el número de sèrie ha d'haver augmentat.
3. Al DNS2, actualitzeu la zona (F5). Gràcies a la notificació, en pocs segons ha d'aparèixer el registre `prova`. Si no apareix, feu **Transfereix des del servidor principal**.
4. Repetiu la prova E.1: els números de sèrie tornen a coincidir.

> 📷 **Captura 7:** registre nou al DNS1, el mateix registre al DNS2 i els números de sèrie actualitzats.

### E.3 El secundari respon als clients

Des del CLIENT, pregunteu directament al servidor secundari:

```
nslookup prova.smx.local 192.168.50.11
nslookup dns2.smx.local 192.168.50.11
```

La resposta ha de donar la IP correcta i **no** ha de mostrar el missatge «Resposta no autoritzada», perquè el DNS2 és servidor autoritzat de la zona.

> 📷 **Captura 8:** consultes del client al DNS2 amb resposta correcta.

### E.4 Tolerància a fallades

1. Atureu el servei DNS del principal (`Stop-Service DNS` al DNS1) o apagueu l'escriptori DNS1 des d'IsardVDI.
2. Al CLIENT, buideu la memòria cau amb `ipconfig /flushdns` i feu `ping prova.smx.local`.
3. El nom s'ha de continuar resolent gràcies al DNS alternatiu (DNS2).
4. Torneu a engegar el servei o l'escriptori DNS1.

> 📷 **Captura 9:** client resolent noms amb el DNS1 aturat.

### E.5 Les transferències estan restringides

Des del CLIENT, intenteu descarregar la zona sencera del principal. Com que el client no és a la llista de servidors de noms, la transferència s'ha de **rebutjar**:

```
nslookup
> server 192.168.50.10
> ls -d smx.local
```

Expliqueu el resultat i per què és important des del punt de vista de la seguretat.

> 📷 **Captura 10:** intent de transferència rebutjat des del client.

### E.6 Registre d'esdeveniments

Al DNS2, obriu el **Visor d'esdeveniments** › Registres d'aplicacions i serveis › **DNS Server** i localitzeu els esdeveniments que indiquen que la zona s'ha transferit i carregat correctament.

> 📷 **Captura 11:** esdeveniment de transferència de zona correcta al DNS2.

## 8. Preguntes

1. Quina diferència hi ha entre una transferència **AXFR** i una **IXFR**?
2. Per què el servidor secundari no pot ser un clon del servidor de la pràctica 1?
3. Què passaria si a la pestanya Transferències de zona escollíssiu **A qualsevol servidor**?
4. Per a què serveix el número de sèrie del registre SOA? I la notificació (NOTIFY)?
5. Què passa amb la zona del secundari si el principal està aturat més temps que el valor **Caduca després de** del SOA?

## 9. Lliurament

Lliureu un únic document (PDF) que inclogui:

- Portada amb nom, curs i títol de la pràctica.
- Totes les captures marcades en aquest enunciat (de la 1 a la 11), numerades i amb una breu explicació de què s'hi veu. A les captures s'ha de poder veure el nom de l'escriptori d'IsardVDI o el nom de l'equip.
- La descripció del mètode de comprovació (Part E) amb el resultat de cada prova.
- Les respostes a les preguntes de l'apartat 8.
- Una breu conclusió: problemes trobats i com els heu resolt.

## Recurs de suport

Podeu consultar aquest vídeo de 10 minuts on es mostra el procediment a Windows Server. No es fa amb IsardVDI, però la configuració del DNS és la mateixa:

https://www.youtube.com/watch?v=1a1pOE3ah14
