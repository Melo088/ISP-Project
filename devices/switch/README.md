# Infraestructura Red GPON - Switch Cisco SG350X-24

## Introducción

Este documento detalla la configuración, funcionalidades y rol del **Switch Cisco SG350X-24** dentro de la infraestructura de red GPON implementada en el proyecto. 
El switch actúa como elemento central de conmutación que interconecta los diferentes segmentos de red: la OLT con clientes GPON, el router interno, y los servidores.

### Identificación del Equipo

- **Modelo**: Cisco SG350X-24
- **Hostname**: switch887629
- **Versión Firmware**: v2.4.5.71 / RTESLA2.4.5_930_181_144
- **CLI Version**: v1.0

## Especificaciones Técnicas

El Cisco SG350X-24 es un switch administrable apilable de nivel empresarial con las siguientes características:

### Hardware

- **Puertos de red**: 24 puertos Gigabit Ethernet 10/100/1000
- **Puertos de uplink**: 4 puertos 10 Gigabit Ethernet (2x 10GBase-T/SFP+ combo + 2x SFP+)
- **Capacidad de conmutación**: 128 Gbps
- **Rendimiento de reenvío**: 95.23 Mpps (paquetes de 64 bytes)
- **Tabla de direcciones MAC**: 16,000 entradas
- **Memoria RAM**: 512 MB
- **Memoria Flash**: 256 MB
- **Buffer de paquetes**: 1.5 MB
- **Factor de forma**: Rack-mountable, 1RU

### Capacidades de Software

- **VLANs activas**: Hasta 4,094 VLANs simultáneas
- **Rutas estáticas**: Hasta 990 rutas IPv4 y 245 rutas IPv6
- **Interfaces IP**: Hasta 128 interfaces IPv4 y 106 interfaces IPv6
- **Routing Layer 3**: Enrutamiento estático IPv4/IPv6
- **Protocolos soportados**: 802.1Q, 802.1X, VLAN routing, SNMP, RADIUS, SNTP, SSH

## Topología de Red

El switch se encuentra en el núcleo de la infraestructura y conecta los siguientes elementos:

```
Internet
   │
   └─► Router Externo (MikroTik RB 3011)
          │ eth1
          └─► Router Interno (MikroTik CCR2004)
                 │ eth2
                 └─► Switch SG350X-24 (G1/0/2)
                        ├─► G1/0/1  → OLT EA5800-X2 → Splitter → ONTs (VLAN 100)
                        ├─► G1/0/10 → Servidor Ubuntu 24.04 (VLAN 20)
                        ├─► G1/0/9  → Web Server (VLAN 50)
                        ├─► G1/0/8  → Servicios Core (VLAN 20)
                        └─► G1/0/11 → Servicios Red (VLAN 40)
```

## Configuración de VLANs

El switch implementa una segmentación de red mediante VLANs para separar el tráfico por función:

### VLAN 10 - Gestión

- **Propósito**: Red de administración del switch
- **Dirección IP**: 192.168.10.253/24
- **Gateway predeterminado**: 192.168.10.1
- **Función**: Acceso SSH, Web UI, SNMP para gestión del equipo

### VLAN 20 - Servicios Core

- **Propósito**: Servicios de red 
- **Puertos asignados**: 
  - G1/0/8: Servicios adicionales
  - G1/0/10: Servidor principal (VMs Ubuntu)

### VLAN 40 - Servicios Correo
- **Puerto asignado**: G1/0/11

### VLAN 50 - Web

- **Propósito**: Servicios web y aplicaciones HTTP/HTTPS
- **Puerto asignado**: G1/0/9 (WebServer)
- **Servicios**: Caddy.

### VLAN 100 - Clientes GPON

- **Propósito**: Red de acceso para usuarios finales conectados vía GPON
- **Rango IPv4**: 192.168.100.100 - 192.168.100.200
- **Puerto troncal**: G1/0/1 (conectado a OLT)
- **Puerto de prueba**: G1/0/15 (acceso directo para testing)
- **Dispositivos**: ONTs Huawei EchoLife EG8145V5

## Configuración de Interfaces

### Interfaces Troncales (Trunk)

Las interfaces troncales transportan múltiples VLANs usando etiquetado 802.1Q:

#### GigabitEthernet1/0/1 - OLT_CLIENTES_GPON

```
interface GigabitEthernet1/0/1
 description OLT_CLIENTES_GPON
 spanning-tree link-type point-to-point
 switchport mode trunk
```

- **Función**: Conexión a la OLT EA5800-X2
- **Tráfico principal**: VLAN 100 (clientes GPON)

#### GigabitEthernet1/0/2 - ROUTER_LAN

```
interface GigabitEthernet1/0/2
 description ROUTER_LAN
 spanning-tree link-type point-to-point
 switchport mode trunk
 macro description "switch"
 macro auto smartport dynamic_type switch
```

- **Función**: Interconexión con Router Interno MikroTik
- **Smartport**: Configuración automática habilitada
- **Propósito**: Gateway hacia Internet y enrutamiento inter-VLAN

### Interfaces de Acceso (Access)

Las interfaces de acceso pertenecen a una única VLAN sin etiquetado:

#### GigabitEthernet1/0/8 - Servicios Core

```
interface GigabitEthernet1/0/8
 switchport access vlan 20
```

- **VLAN**: 20 (Servicios Core)

#### GigabitEthernet1/0/9 - WebServer

```
interface GigabitEthernet1/0/9
 description WebServer
 switchport access vlan 50
```

- **VLAN**: 50 (Web)
- **Uso**: Servidor web con Caddy
- **Servicios**: HTTP/HTTPS

#### GigabitEthernet1/0/10 - SERVIDOR

```
interface GigabitEthernet1/0/10
 description SERVIDOR
 switchport access vlan 20
 no macro auto smartport
```

- **VLAN**: 20 (Servicios Core)
- **Uso**: Servidor principal con VMs Ubuntu

#### GigabitEthernet1/0/11 - Servicios de Correo

```
interface GigabitEthernet1/0/11
 switchport access vlan 40
```

#### GigabitEthernet1/0/15 - Puerto de Pruebas GPON

```
interface GigabitEthernet1/0/15
 switchport access vlan 100
```

- **VLAN**: 100 (Clientes GPON)
- **Uso**: Puerto de acceso directo para pruebas sin pasar por la OLT

## Funcionalidades Implementadas

### SNMP - Simple Network Management Protocol

El switch tiene SNMP habilitado para monitoreo remoto:

```
snmp-server server
snmp-server community gponSwitch ro 192.168.20.40 mask 255.255.255.0 view Default
snmp-server group gponSwitch v2 read All
```

- **Versión**: SNMPv2c
- **Community**: gponSwitch (solo lectura)
- **Host autorizado**: 192.168.20.40 (servidor de monitoreo en VLAN 20)
- **Máscara**: 255.255.255.0
- **Propósito**: Integración con LibreNMS u otras herramientas de monitoreo
- **Información disponible**: Estado de puertos, estadísticas de tráfico, temperatura, errores

### SNTP - Simple Network Time Protocol

Sincronización de tiempo configurada:

```
sntp server 192.168.20.60
```

- **Servidor NTP**: 192.168.20.60 (servidor en VLAN 20)
- **Propósito**: Sincronización horaria para logs y eventos
- **Importancia**: Timestamps precisos para troubleshooting y correlación de eventos

### SSH y Seguridad

```
encrypted ip ssh-client key rsa key-pair
```

- **Protocolo**: SSH habilitado con claves RSA
- **Acceso**: Solo desde VLAN 10 (Gestión)
- **Seguridad**: Autenticación mediante clave pública/privada

### Spanning Tree Protocol

```
spanning-tree link-type point-to-point
```

- **Protocolo**: STP/RSTP habilitado
- **Configuración**: Enlaces punto a punto en puertos trunk
- **Propósito**: Prevención de bucles de red
- **Estado**: Activo en G1/0/1, G1/0/2, G1/0/17, Ten1/0/1

## Enrutamiento y Gateway

### Gateway Predeterminado

```
ip default-gateway 192.168.10.1
```

- **Gateway**: 192.168.10.1 (Router Interno MikroTik)
- **Función**: Salida para gestión remota del switch
- **VLAN**: 10 (Gestión)

