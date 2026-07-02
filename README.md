# VPN Client-to-Site — L2TP sobre IPSec (IKEv1)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es L2TP/IPSec?](#21-qué-es-l2tpipsec)
   - [Arquitectura del protocolo L2TP/IPSec](#22-arquitectura-del-protocolo-l2tpipsec)
   - [Flujo de establecimiento de la conexión](#23-flujo-de-establecimiento-de-la-conexión)
   - [Client-to-Site vs. Site-to-Site](#24-client-to-site-vs-site-to-site)
   - [Parámetros del laboratorio](#25-parámetros-del-laboratorio)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router ISP](#41-router-isp)
   - [Router R1 — Red del Cliente](#42-router-r1--red-del-cliente)
   - [Router R2 — Servidor L2TP/IPSec](#43-router-r2--servidor-l2tpipsec)
   - [Configuración del Cliente Linux](#44-configuración-del-cliente-linux)
5. [Verificación de la Conexión](#5-verificación-de-la-conexión)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Client-to-Site punto a multipunto utilizando L2TP sobre IPSec con IKEv1** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración de un servidor L2TP/IPSec en Cisco IOS mediante VPDN (Virtual Private Dial-up Networking), incluyendo la autenticación de usuarios con MS-CHAPv2/CHAP a través de la base de datos local del router.
* El uso de IPSec en **modo transporte** (no túnel) para cifrar el tráfico L2TP — L2TP provee la encapsulación PPP y la asignación de IP al cliente; IPSec provee el cifrado de ese canal L2TP.
* La asignación dinámica de direcciones IP a los clientes conectados mediante un pool local (`20.25.37.100–20.25.37.110`), permitiendo que el cliente acceda a la red interna del servidor como si estuviera conectado localmente.
* La verificación de conectividad desde el cliente remoto hacia la red interna (`20.25.37.0/25`) a través del túnel L2TP/IPSec establecido con R2.

---

## 2. Marco Teórico

### 2.1 ¿Qué es L2TP/IPSec?

L2TP (Layer 2 Tunneling Protocol — RFC 2661) es un protocolo de tunelización de **Capa 2** que encapsula tramas PPP dentro de paquetes UDP (puerto 1701). Por sí solo, L2TP **no cifra** el contenido — solo crea el túnel. Por eso se combina con IPSec para proteger el canal.

La combinación L2TP/IPSec resulta en una VPN que ofrece:

* **Autenticación de usuario** (PPP dentro de L2TP): MS-CHAPv2, CHAP, PAP — valida quién es el usuario.
* **Autenticación del túnel** (IPSec): Pre-Shared Key o certificados — valida que el servidor VPN es legítimo.
* **Cifrado** (IPSec ESP): AES, 3DES — protege el contenido del tráfico PPP/L2TP.

L2TP/IPSec es el protocolo VPN **nativo en Windows, macOS, iOS y Android** — no requiere instalar software cliente adicional, lo que lo hace ideal para VPNs de acceso remoto corporativo.

### 2.2 Arquitectura del protocolo L2TP/IPSec

```
CLIENTE                                          SERVIDOR (R2)
─────────────────────────────────────────────────────────────
  │                                                      │
  │  ┌─────────────────────────────────────────────┐    │
  │  │           IPSec ESP (cifrado)               │    │
  │  │  ┌──────────────────────────────────────┐   │    │
  │  │  │     L2TP (túnel UDP 1701)            │   │    │
  │  │  │  ┌───────────────────────────────┐   │   │    │
  │  │  │  │   PPP (autenticación + datos) │   │   │    │
  │  │  │  │  ┌────────────────────────┐   │   │   │    │
  │  │  │  │  │   IP del cliente       │   │   │   │    │
  │  │  │  │  │   (20.25.37.100)       │   │   │   │    │
  │  │  │  │  └────────────────────────┘   │   │   │    │
  │  │  │  └───────────────────────────────┘   │   │    │
  │  │  └──────────────────────────────────────┘   │    │
  │  └─────────────────────────────────────────────┘    │
  │                                                      │

Puertos involucrados:
  UDP 500  → IKE (negociación IPSec)
  UDP 4500 → NAT-T (si hay NAT entre cliente y servidor)
  UDP 1701 → L2TP (dentro del túnel IPSec, no visible en WAN)
```

### 2.3 Flujo de establecimiento de la conexión

La conexión L2TP/IPSec se establece en este orden estricto:

| Paso | Protocolo | Descripción |
|---|---|---|
| 1 | IKE Fase 1 (ISAKMP) | Autenticación del túnel con PSK — establece canal IKE seguro |
| 2 | IKE Fase 2 (IPSec SA) | Negocia parámetros ESP en modo transporte para proteger UDP 1701 |
| 3 | L2TP | Dentro del canal IPSec, el cliente abre el túnel L2TP al servidor |
| 4 | PPP / MS-CHAPv2 | Autenticación del **usuario** con usuario y contraseña |
| 5 | IPCP | El servidor asigna una IP al cliente desde el pool VPN |
| 6 | Datos | El tráfico del cliente viaja cifrado por IPSec → L2TP → PPP |

IPSec se negocia **primero**, antes de que L2TP empiece — esto garantiza que las credenciales del usuario (paso 4) nunca viajen sin cifrar.

### 2.4 Client-to-Site vs. Site-to-Site

| Característica | Client-to-Site (este lab) | Site-to-Site (labs anteriores) |
|---|---|---|
| **Extremos** | Un usuario individual + un router/servidor | Dos routers fijos |
| **IP del cliente** | Dinámica — asignada por el servidor (pool) | Fija — configurada estáticamente |
| **Autenticación usuario** | Usuario/contraseña (MS-CHAPv2) | Solo autenticación del equipo (PSK/certificado) |
| **Protocolo** | L2TP/IPSec, SSTP, IKEv2 con EAP | IPSec IKEv1/IKEv2 con PSK |
| **Escalabilidad** | Muchos usuarios con un solo servidor | N sitios fijos |
| **Cliente** | Windows, Linux, macOS (nativo) | Router Cisco, firewall |

### 2.5 Parámetros del laboratorio

| Parámetro | Valor | Descripción |
|---|---|---|
| **Cifrado IKE** | 3DES | Canal de negociación IKE |
| **Hash IKE** | SHA-1 | Integridad de mensajes IKE |
| **Grupo DH** | Grupo 2 (1024-bit) | Intercambio de clave |
| **Autenticación PSK** | `MiClaveVPN123` | Valida que el servidor es legítimo |
| **Cifrado ESP** | 3DES | Cifrado del tráfico L2TP |
| **Hash ESP** | MD5-HMAC | Integridad del tráfico L2TP |
| **Modo IPSec** | Transport | L2TP encapsula, IPSec solo cifra |
| **Autenticación usuario** | MS-CHAPv2 / CHAP | Valida usuario/contraseña del cliente |
| **Usuario VPN** | `cliente1` | Nombre de usuario configurado localmente |
| **Contraseña VPN** | `cisco123` | Contraseña del usuario |
| **Pool de IPs** | `20.25.37.100 – 20.25.37.110` | IPs asignadas a clientes conectados |
| **Protocolo L2TP** | UDP 1701 (dentro de IPSec) | Tunelización de capa 2 |

> **Nota de seguridad:** Este laboratorio usa 3DES y MD5 por compatibilidad con clientes L2TP nativos de Windows y Linux. En producción se recomienda migrar a IKEv2 con AES-256 y SHA-256 para mayor seguridad.

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio implementa una VPN Client-to-Site donde el cliente (máquina Linux/Windows en la LAN de R1) se conecta al servidor VPN (R2) a través del ISP. Una vez conectado, el cliente recibe una IP del pool VPN y puede acceder a la red interna de R2 (`20.25.37.0/25`).

```
          LAN Cliente                 [ ISP ]                   LAN Servidor
         20.25.37.128/25          192.168.1.0/30            20.25.37.0/25
                                  192.168.1.4/30
              │                        │                          │
    ┌─────────┴─────────┐    ┌─────────┴─────────┐    ┌─────────┴─────────┐
    │  Router R1        │    │  Router ISP        │    │  Router R2        │
    │                   │    │                    │    │  (Servidor L2TP)  │
    │  e0/0: 192.168.1.2│────│e0/0: 192.168.1.1  │    │                   │
    │  e0/1: 20.25.37.129    │e0/1: 192.168.1.5  │────│Gi1: 192.168.1.6   │
    └─────────┬─────────┘    └────────────────────┘    └─────────┬─────────┘
              │ e0/1                                              │ Gi2
              │                                                   │
    ┌─────────┴─────────┐                              ┌─────────┴─────────┐
    │       SW1         │                              │       SW2         │
    └─────────┬─────────┘                              └─────────┬─────────┘
              │ e0/1                                              │ e0/2
              │ e0                                                │ eth0
    ┌─────────┴─────────┐                              ┌─────────┴─────────┐
    │     Cliente       │                              │       PC1         │
    │  20.25.37.130/25  │                              │  20.25.37.2/25    │
    └───────────────────┘                              └───────────────────┘

  ══════════════════════════════════════════════════════════════════
  Flujo de la conexión L2TP/IPSec:
    1. Cliente inicia VPN hacia 192.168.1.6 (IP WAN de R2).
    2. IKEv1 negocia IPSec SA (3DES/SHA/PSK MiClaveVPN123).
    3. L2TP abre túnel UDP 1701 dentro del canal IPSec.
    4. PPP autentica al usuario: cliente1 / cisco123 (MS-CHAPv2).
    5. R2 asigna IP al cliente desde el pool: 20.25.37.100.
    6. Cliente puede ahora alcanzar PC1 (20.25.37.2) a través
       del túnel, como si estuviera en la red interna de R2.
  ══════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **ISP** | Cisco IOS (Router) | e0/0 | 192.168.1.1 | /30 | — | Enlace hacia R1 |
| | | e0/1 | 192.168.1.5 | /30 | — | Enlace hacia R2 |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.2 | /30 | 192.168.1.1 | Gateway WAN — red del cliente |
| | | e0/1 | 20.25.37.129 | /25 | — | Gateway LAN — red del cliente |
| **R2** | Cisco IOS (Router) | Gi1 | 192.168.1.6 | /30 | 192.168.1.5 | Gateway WAN — servidor L2TP |
| | | Gi2 | 20.25.37.1 | /25 | — | Gateway LAN — red interna |
| | | Virtual-Template1 | `ip unnumbered Gi2` | — | — | Interfaz virtual L2TP |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN cliente |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN servidor |
| **Cliente** | Linux / Windows | e0 | 20.25.37.130 | /25 | 20.25.37.129 | Cliente VPN |
| | | ppp0 (VPN activa) | 20.25.37.100 | /25 | — | IP asignada por el pool VPN |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.2 | /25 | 20.25.37.1 | Host en red interna de R2 |

---

## 4. Scripts de Configuración

### 4.1 Router ISP

```cisco
! ══════════════════════════════════════════════════════
! ISP | Router de tránsito
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R-ISP

interface Ethernet0/0
 description hacia-R1
 ip address 192.168.1.1 255.255.255.252
 no shutdown

interface Ethernet0/1
 description hacia-R2
 ip address 192.168.1.5 255.255.255.252
 no shutdown
```

### 4.2 Router R1 — Red del Cliente

```cisco
! ══════════════════════════════════════════════════════
! R1 — Red del Cliente | Solo enrutamiento
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.2 255.255.255.252
 no shutdown

interface Ethernet0/1
 description LAN-Cliente
 ip address 20.25.37.129 255.255.255.128
 no shutdown

! Ruta por defecto hacia Internet
ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

### 4.3 Router R2 — Servidor L2TP/IPSec

```cisco
! ══════════════════════════════════════════════════════
! R2 — Servidor L2TP/IPSec | IKEv1
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces físicas ────────────────────────────────
interface GigabitEthernet1
 description WAN-hacia-ISP
 ip address 192.168.1.6 255.255.255.252
 no shutdown

interface GigabitEthernet2
 description LAN-Interna
 ip address 20.25.37.1 255.255.255.128
 no shutdown

! ─── Rutas ─────────────────────────────────────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.5
! Ruta de retorno hacia la LAN del Cliente (para que R2 sepa
! como responder al cliente una vez conectado vía VPN)
ip route 20.25.37.128 255.255.255.128 192.168.1.2

! ══════════════════════════════════════════════════════
! PASO 1: AAA — autenticación de usuarios PPP
! ══════════════════════════════════════════════════════
! aaa new-model habilita el framework de autenticación.
! La política "default" aplica a todas las conexiones PPP.
aaa new-model
aaa authentication ppp default local

! ─── Usuarios locales ──────────────────────────────────
username cliente1 password cisco123

! ══════════════════════════════════════════════════════
! PASO 2: IKEv1 — protege el canal L2TP con IPSec
! ══════════════════════════════════════════════════════

! ISAKMP Policy — IKE Fase 1
crypto isakmp policy 10
 encryption 3des
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

! PSK wildcard — acepta conexiones desde cualquier IP de cliente
! (necesario porque los clientes pueden tener IPs dinámicas)
crypto isakmp key MiClaveVPN123 address 0.0.0.0 0.0.0.0

! Keepalive para detectar clientes desconectados detrás de NAT
crypto isakmp nat keepalive 20

! ══════════════════════════════════════════════════════
! PASO 3: IPSec — cifra el tráfico L2TP (UDP 1701)
! ══════════════════════════════════════════════════════

! Transform Set en modo TRANSPORTE — L2TP ya encapsula,
! IPSec solo necesita cifrar el payload UDP.
crypto ipsec transform-set TS-L2TP esp-3des esp-md5-hmac
 mode transport

! Dynamic Map — acepta SAs de peers con IP dinámica
crypto dynamic-map DMAP 10
 set transform-set TS-L2TP

! Crypto Map usando el Dynamic Map
crypto map CMAP 10 ipsec-isakmp dynamic DMAP

! Aplicar el Crypto Map a la interfaz WAN
interface GigabitEthernet1
 crypto map CMAP

! ══════════════════════════════════════════════════════
! PASO 4: VPDN — servidor L2TP
! ══════════════════════════════════════════════════════

vpdn enable

vpdn-group L2TP-CLIENTES
 accept-dialin
  protocol l2tp
  virtual-template 1          ! Usa la plantilla de interfaz virtual
 no l2tp tunnel authentication ! Sin autenticación adicional del túnel L2TP

! ══════════════════════════════════════════════════════
! PASO 5: Pool de IPs y Virtual-Template
! ══════════════════════════════════════════════════════

! Pool de direcciones asignadas a los clientes VPN
ip local pool POOL-VPN 20.25.37.100 20.25.37.110

! Virtual-Template — plantilla para crear interfaces
! virtuales dinámicas por cada cliente conectado
interface Virtual-Template1
 ip unnumbered GigabitEthernet2   ! Usa la IP de Gi2 (20.25.37.1)
 peer default ip address pool POOL-VPN
 ppp authentication ms-chap-v2 chap
```

### 4.4 Configuración del Cliente Linux

El cliente Linux usa `xl2tpd` y `strongSwan` (o `ipsec-tools`) para conectarse al servidor L2TP/IPSec.

**Instalar dependencias:**

```bash
sudo apt-get install xl2tpd strongswan -y
```

**Archivo `/etc/ipsec.conf`:**

```bash
config setup
    charondebug="ike 1, knl 1, cfg 0"

conn L2TP-VPN
    authby=secret
    auto=add
    keyexchange=ikev1
    left=%defaultroute
    leftprotoport=17/1701
    right=192.168.1.6
    rightprotoport=17/1701
    type=transport
    ike=3des-sha1-modp1024
    esp=3des-md5
```

**Archivo `/etc/ipsec.secrets`:**

```bash
%any 192.168.1.6 : PSK "MiClaveVPN123"
```

**Archivo `/etc/xl2tpd/xl2tpd.conf`:**

```bash
[lac vpn-servidor]
lns = 192.168.1.6
ppp debug = yes
pppoptfile = /etc/ppp/options.l2tpd.client
length bit = yes
```

**Archivo `/etc/ppp/options.l2tpd.client`:**

```bash
ipcp-accept-local
ipcp-accept-remote
refuse-eap
require-mschap-v2
noccp
noauth
mtu 1280
mru 1280
noipdefault
defaultroute
usepeerdns
connect-delay 5000
name cliente1
password cisco123
```

**Iniciar la conexión:**

```bash
sudo ipsec restart
sudo service xl2tpd restart
sudo ipsec up L2TP-VPN
echo "c vpn-servidor" | sudo tee /var/run/xl2tpd/l2tp-control
```

**Verificar la interfaz VPN asignada:**

```bash
ip addr show ppp0
```

---

## 5. Verificación de la Conexión

### 5.1 Verificar el estado IKEv1 en R2

```cisco
R2# show crypto isakmp sa
```

*Salida esperada con cliente conectado:*

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
192.168.1.6     192.168.1.2     QM_IDLE        1001    ACTIVE
```

> `QM_IDLE` confirma que la Fase 1 IPSec completó. Si la IP de origen es diferente (el cliente tiene NAT), aparecerá la IP del NAT.

---

### 5.2 Verificar las IPSec SAs

```cisco
R2# show crypto ipsec sa
```

*Salida esperada (fragmento):*

```
interface: GigabitEthernet1
    Crypto map tag: CMAP, local addr 192.168.1.6

   local  ident (addr/mask/prot/port): (192.168.1.6/255.255.255.255/17/1701)
   remote ident (addr/mask/prot/port): (192.168.1.2/255.255.255.255/17/1701)

    #pkts encaps: 45, #pkts encrypt: 45, #pkts digest: 45
    #pkts decaps: 45, #pkts decrypt: 45, #pkts verify: 45
```

> Los identifiers muestran `protocolo 17 / puerto 1701` — confirma que IPSec está protegiendo específicamente el tráfico L2TP (UDP 1701).

---

### 5.3 Verificar sesiones VPDN activas

```cisco
R2# show vpdn session
```

*Salida esperada:*

```
L2TP Session Information Total tunnels 1 sessions 1

LocID RemID TunID Intf    Username    State     Last Chg
1     1     1     Vi1     cliente1    est       00:02:15
```

> `State: est` (established) confirma que la sesión L2TP está activa. `Vi1` es la Virtual-Access interface creada dinámicamente para este cliente.

---

### 5.4 Verificar la interfaz virtual asignada al cliente

```cisco
R2# show ip interface brief | include Virtual
```

*Salida esperada:*

```
Virtual-Access1        20.25.37.1      YES unset  up   up
Virtual-Template1      20.25.37.1      YES unset  up   down
```

---

### 5.5 Verificar la IP asignada al cliente desde el pool

```cisco
R2# show ip local pool POOL-VPN
```

*Salida esperada:*

```
Pool          Begin           End             Free  In use
POOL-VPN      20.25.37.100    20.25.37.110    10    1
```

> `In use: 1` confirma que una IP del pool fue asignada al cliente conectado.

---

### 5.6 Verificar conectividad del cliente hacia la red interna

Desde el **Cliente Linux** (con la VPN activa):

```bash
ping 20.25.37.2 -c 4
```

*Resultado esperado:*

```
PING 20.25.37.2 (20.25.37.2) 56(84) bytes of data.
64 bytes from 20.25.37.2: icmp_seq=1 ttl=63 time=X ms
64 bytes from 20.25.37.2: icmp_seq=2 ttl=63 time=X ms
```

> TTL=63 (en vez de 64) indica que el paquete pasó por un router (R2) para llegar a PC1. La conectividad desde un cliente remoto a la red interna confirma el funcionamiento completo de la VPN.

---

### 5.7 Tabla de comandos de verificación

| Comando | Dónde | Qué muestra |
|---|---|---|
| `show crypto isakmp sa` | R2 | SA IKE Fase 1 — debe ser `QM_IDLE` |
| `show crypto ipsec sa` | R2 | SA IPSec — protocolo 17/1701 en los identifiers |
| `show vpdn session` | R2 | Sesiones L2TP activas y usuarios autenticados |
| `show vpdn tunnel` | R2 | Información del túnel L2TP establecido |
| `show ip local pool POOL-VPN` | R2 | IPs del pool en uso |
| `show ppp all` | R2 | Estado de las sesiones PPP activas |
| `ip addr show ppp0` | Cliente Linux | IP asignada al cliente por el servidor |
| `ping 20.25.37.2 -c 4` | Cliente Linux | Conectividad hacia la red interna de R2 |

---

## 6. Capturas de Pantalla

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_config_r2_isakmp.png`](screenshots/02_config_r2_isakmp.png) | Consola de R2 mostrando la `crypto isakmp policy 10` con 3DES/SHA/PSK y el `crypto isakmp key` con wildcard `0.0.0.0`. |
| 3 | [`03_config_r2_vpdn.png`](screenshots/03_config_r2_vpdn.png) | Consola de R2 mostrando el `vpdn-group L2TP-CLIENTES`, el `Virtual-Template1` y el `ip local pool POOL-VPN`. |
| 4 | [`04_cliente_linux_config.png`](screenshots/04_cliente_linux_config.png) | Terminal del Cliente mostrando los archivos de configuración `ipsec.conf` y `xl2tpd.conf` aplicados. |
| 5 | [`05_cliente_conexion_inicio.png`](screenshots/05_cliente_conexion_inicio.png) | Terminal del Cliente mostrando los comandos de inicio de la VPN (`ipsec up` y escritura en `l2tp-control`). |
| 6 | [`06_isakmp_sa_qmidle.png`](screenshots/06_isakmp_sa_qmidle.png) | Salida de `show crypto isakmp sa` en R2 mostrando estado `QM_IDLE` con la IP del cliente. |
| 7 | [`07_vpdn_session_est.png`](screenshots/07_vpdn_session_est.png) | Salida de `show vpdn session` en R2 mostrando `cliente1` con estado `est` y la interfaz `Vi1`. |
| 8 | [`08_pool_in_use.png`](screenshots/08_pool_in_use.png) | Salida de `show ip local pool POOL-VPN` mostrando `In use: 1` — IP asignada al cliente. |
| 9 | [`09_ppp0_ip_asignada.png`](screenshots/09_ppp0_ip_asignada.png) | Terminal del Cliente mostrando `ip addr show ppp0` con la IP `20.25.37.100` asignada por el servidor. |
| 10 | [`10_ping_red_interna.png`](screenshots/10_ping_red_interna.png) | Ping exitoso desde el Cliente hacia PC1 (`20.25.37.2`) con la VPN activa, TTL=63. |

---

## 7. Consideraciones de Seguridad

### 7.1 Debilidades de esta configuración

* **3DES y MD5:** Algoritmos considerados legacy. 3DES tiene vulnerabilidades conocidas (SWEET32) y MD5 está comprometido criptográficamente. Se usan aquí por compatibilidad con clientes L2TP nativos de Windows/Linux que no siempre soportan AES en IKEv1 L2TP.
* **PSK wildcard:** El servidor acepta conexiones de cualquier IP con la PSK correcta — cualquier persona que conozca `MiClaveVPN123` puede iniciar la negociación IPSec.
* **MS-CHAPv2:** Vulnerable a ataques de diccionario offline si se captura el handshake. En producción se reemplaza por EAP-TLS con certificados.
* **Grupo DH 2:** 1024-bit — considerado insuficiente desde 2015. El mínimo recomendado hoy es Grupo 14 (2048-bit).

### 7.2 Migración recomendada para producción

Para un entorno de producción, la alternativa moderna es **IKEv2 con EAP** en lugar de L2TP/IPSec con IKEv1:

```cisco
! IKEv2 con EAP — reemplaza todo el bloque L2TP/IKEv1
crypto ikev2 proposal PROP-ACCESO
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 profile PROF-CLIENTES
 match identity remote any
 authentication local rsa-sig    ! Certificado del servidor
 authentication remote eap       ! Usuario/contraseña via EAP
 aaa authentication eap default local
```

Esta configuración es la base de **FlexVPN** y es soportada nativamente por iOS, macOS y Windows 10/11 con el cliente VPN integrado.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Configuración de R2 (servidor L2TP) mostrando los bloques IKEv1, IPSec, VPDN y Virtual-Template.
* ✅ Inicio de la conexión VPN desde el cliente Linux.
* ✅ `show crypto isakmp sa` en R2 mostrando `QM_IDLE`.
* ✅ `show vpdn session` mostrando `cliente1` en estado `est`.
* ✅ `ip addr show ppp0` en el cliente mostrando la IP asignada del pool.
* ✅ Ping exitoso desde el cliente hacia PC1 (`20.25.37.2`) con la VPN activa.

---

## 9. Referencias

* Townsley, W. et al. (1999). *RFC 2661 — Layer Two Tunneling Protocol (L2TP)*. IETF.
* Harkins, D. & Carrel, D. (1998). *RFC 2409 — The Internet Key Exchange (IKEv1)*. IETF.
* Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide — L2TP over IPSec*.
* Cisco Systems. (2024). *Cisco IOS VPDN Configuration Guide*.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
