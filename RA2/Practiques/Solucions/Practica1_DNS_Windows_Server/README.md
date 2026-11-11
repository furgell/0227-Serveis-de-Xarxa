# Solució · Pràctica 1: Servidor DNS a Windows Server

Valors fets servir: **X = 2**, servidor **SERVERURGELL** (192.168.2.1), client PC01 (192.168.2.10).

> La Part 5 (comprovacions des del client Windows 10) encara no està documentada.

## Part 1 · Preparar el servidor

| Captura | Què es veu |
|---|---|
| [01](captures/01_propietats_sistema.jpg) | Propietats del sistema abans de canviar el nom (WIN-…). |
| [02](captures/02_canvi_nom.jpg) | Canvi del nom de l'equip a SERVERURGELL. El noVNC no envia majúscules; Windows desa el nom NetBIOS en majúscules igualment. |
| [03](captures/03_connexions_xarxa.jpg) | `ncpa.cpl`: interfície 1 (Isard, DHCP) i interfície 2 (*Red no identificada*). |
| [04](captures/04_ip_estatica.jpg) | TCP/IPv4 de la interfície 2: 192.168.2.1/24, sense porta d'enllaç, DNS 192.168.2.1 i 127.0.0.1. |
| [05](captures/05_ipconfig_servidor.jpg) | `hostname` i `ipconfig`: la interfície 2 mostra 192.168.2.1. |
| [06](captures/06_tallafoc_estat.jpg) | `wf.msc`: en aquesta plantilla el tallafoc ja està desactivat en tots els perfils. |
| [07](captures/07_regla_icmpv4.jpg) | Regla *Archivos e impresoras compartidos (petición eco: ICMPv4 de entrada)* habilitada. |
| [08](captures/08_servidor_local.jpg) | Administrador del servidor > Servidor local: nom i IP de la interfície 2. |

## Part 2 · Rol Servidor DNS

| Captura | Què es veu |
|---|---|
| [09](captures/09_tipus_installacio.jpg) | Instal·lació basada en característiques o en rols. |
| [10](captures/10_seleccio_servidor.jpg) | Selecció del servidor serverurgell. |
| [11](captures/11_agregar_caracteristiques.jpg) | Afegir les eines d'administració del DNS. |
| [12](captures/12_rol_dns_marcat.jpg) | Rol *Servidor DNS* marcat. |
| [13](captures/13_confirmacio.jpg) | Confirmació de la instal·lació. |
| [14](captures/14_installacio_correcta.jpg) | *Instalación correcta en serverurgell*. |
| [15](captures/15_administrador_dns.jpg) | Administrador de DNS obert. |
| [16](captures/16_interfaces_dns.jpg) | Pestanya *Interfaces*: el DNS només escolta a 192.168.2.1. |

## Part 3 · Zones

| Captura | Què es veu |
|---|---|
| [17](captures/17_zona_principal.jpg) | Tipus de zona: principal. |
| [18](captures/18_nom_zona_smx.jpg) | Nom de la zona directa: `smx.com`. |
| [19](captures/19_arxiu_zona.jpg) | Arxiu de zona nou `smx.com.dns`. |
| [20](captures/20_actualitzacio_dinamica.jpg) | No admetre actualitzacions dinàmiques. |
| [21](captures/21_resum_zona_directa.jpg) | Resum de l'assistent. |
| [22](captures/22_zona_smx_creada.jpg) | `smx.com`: Primaria estándar, En ejecución. |
| [23](captures/23_inversa_ipv4.jpg) | Zona de cerca inversa per a IPv4. |
| [24](captures/24_id_xarxa_inversa.jpg) | Id. de xarxa 192.168.2 → `2.168.192.in-addr.arpa`. |
| [25](captures/25_resum_zona_inversa.jpg) | Resum de la zona inversa. |
| [26](captures/26_zona_inversa_creada.jpg) | Zona inversa creada i en execució. |

## Part 4 · Registre del servidor i comprovació

| Captura | Què es veu |
|---|---|
| [27](captures/27_host_smxservidor.jpg) | Host nou `smxservidor` → 192.168.2.1 amb el PTR marcat. |
| [28](captures/28_host_creat.jpg) | Registre creat amb èxit. |
| [29](captures/29_registres_zona_directa.jpg) | Zona `smx.com`: SOA, NS, A smxservidor, A pc01, CNAME www. |
| [30](captures/30_registres_ptr.jpg) | Zona inversa: PTR 192.168.2.1 i 192.168.2.10. |
| [31](captures/31_incidencia_dns_isard.jpg) | **Incidència**: `nslookup` pregunta al DNS d'Isard (10.100.1.252) de la interfície 1 i surt *UnKnown* / temps d'espera. |
| [32](captures/32_metrica_interficie2.jpg) | **Solució**: mètrica manual 1 a la interfície 2 (Opciones avanzadas de TCP/IPv4), sense tocar la interfície 1. |
| [33](captures/33_nslookup_servidor.jpg) | `nslookup smxservidor.smx.com` i `nslookup 192.168.2.1`: Servidor smxservidor.smx.com, resultat correcte. |

## Exercicis

| Captura | Què es veu |
|---|---|
| [38](captures/38_ex1_cname_www.jpg) | Ex. 1: àlies (CNAME) `www` → `smxservidor.smx.com`. |
| [39](captures/39_ex1_comprovacio_www.jpg) | Ex. 1: `nslookup www.smx.com` (mostra *Aliases*) i `ping www.smx.com` (0% perduts). |
| [40](captures/40_ex2_host_pc01.jpg) | Ex. 2: host `pc01` → 192.168.2.10 amb PTR. |
| [41](captures/41_ex2_comprovacio_pc01.jpg) | Ex. 2: `nslookup pc01.smx.com` i `nslookup 192.168.2.10`. |
| [42](captures/42_ex3_reenviador_validat.jpg) | Ex. 3: reenviador 8.8.8.8 validat (dns.google). |
| [43](captures/43_ex3_reenviadors_pestanya.jpg) | Ex. 3: pestanya *Reenviadores* amb 8.8.8.8. |
| [44](captures/44_ex3_nslookup_google.jpg) | Ex. 3: `nslookup www.google.com` resolt a través del reenviador (resposta no autoritativa). |

Les comprovacions dels exercicis estan fetes des del servidor.
