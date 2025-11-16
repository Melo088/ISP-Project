# Infraestructura Red GPON - Router Interno

## 1. Información General del Proyecto

### 1.1 Descripción General
Este documento describe la configuración y funcionamiento del **Router Interno MikroTik CCR2004-16G-2S+PC**, el cual actúa como núcleo de enrutamiento y segmentación de red en una infraestructura GPON académica.

### 1.2 Topología de Red

La infraestructura está compuesta por los siguientes equipos:

- **Router Externo**: MikroTik RB 3011 UiAS-RM (salida a Internet)
- **Router Interno**: MikroTik CCR2004-16G-2S+PC (núcleo del proyecto)
- **Switch Core**: Cisco SG350X-24
- **OLT**: EA5800-X2
- **ONT**: Huawei EchoLife EG8145V5
- **Splitter**: FiberHome Celcia
- **Servidor**: Ubuntu Server 24.04 (máquinas virtuales con Vagrant)

### 1.3 Conectividad del Router Interno

El router interno tiene las siguientes conexiones físicas:

- **ether1 (CISCO_SWITCH)**: Conexión al switch Cisco SG350X-24
- **ether2 (ROUTER_ONE)**: Conexión al router externo (uplink a Internet)

A través del switch se conectan:
- **G1/0/1**: OLT (EA5800-X2)
- **G1/0/2**: Router Interno
- **G1/0/10**: Servidor Ubuntu con servicios

---

## 2. Especificaciones Técnicas del Router

### 2.1 Hardware

| Característica | Especificación |
|---------------|----------------|
| **Modelo** | CCR2004-16G-2S+PC |
| **CPU** | AL32400 ARM 64-bit, 4 cores @ 1.2 GHz |
| **RAM** | 4 GB DDR4 |
| **Almacenamiento** | 128 MB NAND |
| **Puertos Ethernet** | 16x Gigabit Ethernet |
| **Puertos SFP+** | 2x 10G SFP+ |
| **Switch Chip** | Marvell 88E6191X (2x chips, 8 puertos c/u) |
| **Consumo** | Max 36W |
| **Sistema Operativo** | RouterOS v7.13.5 |
| **Licencia** | Level 6 |
| **Serial Number** | HGA09ZBSP3T |

---

## 3. Arquitectura de Segmentación de Red

### 3.1 VLANs Configuradas

El router implementa una arquitectura de red segmentada mediante VLANs sobre el trunk hacia el switch Cisco:

| VLAN ID | Nombre | Red IPv4 | Red IPv6 | Propósito |
|---------|--------|----------|----------|-----------|
| **10** | vlan10-Gestion | 192.168.10.0/24 | 2001:db8:10::/64 | Gestión de equipos |
| **20** | vlan20-Core | 192.168.20.0/24 | 2001:db8:20::/64 | Servicios core (DNS, DHCP, NTP) |
| **30** | vlan30-Monitoreo | 192.168.30.0/24 | 2001:db8:30::/64 | Monitoreo (LibreNMS, SNMP) |
| **40** | vlan40-Servicios | 192.168.40.0/24 | 2001:db8:40::/64 | Servicios de aplicación (Mail) |
| **50** | vlan50-Web | 192.168.50.0/24 | 2001:db8:50::/64 | Servidores web (Caddy, Reverse Proxy) |
| **100** | vlan100-GPON | 192.168.100.0/24 | 2001:db8:100::/64 | Clientes GPON (ONTs) |

### 3.2 Interfaces VLAN

Todas las VLANs están configuradas sobre la interfaz física **CISCO_SWITCH (ether1)**, que funciona como trunk 802.1Q.

```
/interface vlan
add interface=CISCO_SWITCH name=vlan10-Gestion vlan-id=10
add interface=CISCO_SWITCH name=vlan20-Core vlan-id=20
add interface=CISCO_SWITCH name=vlan30-Monitoreo vlan-id=30
add interface=CISCO_SWITCH name=vlan40-Servicios vlan-id=40
add interface=CISCO_SWITCH name=vlan50-Web vlan-id=50
add interface=CISCO_SWITCH name=vlan100-GPON vlan-id=100
```

---

## 4. Direccionamiento IP

### 4.1 IPv4

El router actúa como **gateway** para todas las VLANs con la dirección `.1` en cada subred:

```
/ip address
add address=192.168.10.1/24 interface=vlan10-Gestion network=192.168.10.0
add address=192.168.20.1/24 interface=vlan20-Core network=192.168.20.0
add address=192.168.30.1/24 interface=vlan30-Monitoreo network=192.168.30.0
add address=192.168.40.1/24 interface=vlan40-Servicios network=192.168.40.0
add address=192.168.50.1/24 interface=vlan50-Web network=192.168.50.0
add address=192.168.100.1/24 interface=vlan100-GPON network=192.168.100.0
add address=192.168.88.142/24 interface=ROUTER_ONE network=192.168.88.0
```

La interfaz **ROUTER_ONE** (ether2) tiene asignada la IP estática `192.168.88.142/24`, pero también está configurado un cliente DHCP que podría obtener una IP dinámica del router externo.

### 4.2 IPv6

Se implementa direccionamiento IPv6 local único (ULA) con el prefijo `2001:db8::/32`:

```
/ipv6 address
add address=2001:db8:10::1 interface=vlan10-Gestion
add address=2001:db8:20::1 interface=vlan20-Core
add address=2001:db8:30::1 interface=vlan30-Monitoreo
add address=2001:db8:40::1 interface=vlan40-Servicios
add address=2001:db8:50::1 interface=vlan50-Web
add address=2001:db8:100::1 interface=vlan100-GPON
```

### 4.3 Anuncios de Router IPv6 (RA)

Se configuró Router Advertisement en todas las VLANs para autoconfiguración SLAAC:

```
/ipv6 nd
add dns=2001:db8:20::20 interface=vlan100-GPON ra-interval=20s-1m
add dns=2001:db8:20::20 interface=vlan10-Gestion ra-interval=20s-1m
add dns=2001:db8:20::20 interface=vlan20-Core ra-interval=20s-1m
add dns=2001:db8:20::20 interface=vlan30-Monitoreo ra-interval=20s-1m
add dns=2001:db8:20::20 interface=vlan40-Servicios ra-interval=20s-1m
add dns=2001:db8:20::20 interface=vlan50-Web ra-interval=20s-1m
```

- **DNS IPv6**: `2001:db8:20::20` (servidor DNS en VLAN 20)
- **Intervalo RA**: 20 segundos a 1 minuto

---

## 5. DHCP Relay

### 5.1 Arquitectura DHCP

El router **NO** actúa como servidor DHCP. En su lugar, implementa **DHCP Relay** para reenviar las solicitudes DHCP de los clientes hacia el servidor centralizado ubicado en la VLAN 20.

### 5.2 Servidor DHCP Centralizado

- **IP del servidor**: `192.168.20.2` (Kea DHCP Server en Ubuntu)
- **VLANs con relay**: VLAN 100 (GPON)

### 5.3 Configuración del Relay

```
/ip dhcp-relay
add dhcp-server=192.168.20.2 disabled=no interface=vlan100-GPON \
    local-address=192.168.100.1 name=DHCP-VLAN100
```

**Funcionamiento**:
1. Cliente en VLAN 100 envía DHCPDISCOVER (broadcast)
2. Router recibe la solicitud y la reenvía vía unicast a 192.168.20.2
3. Servidor DHCP responde al router
4. Router reenvía la respuesta al cliente
---

## 6. Enrutamiento

### 6.1 Enrutamiento IPv4

#### Ruta por Defecto
```
/ip route
add dst-address=0.0.0.0/0 gateway=192.168.88.1
```

Todo el tráfico destinado a Internet se envía al router externo (192.168.88.1 en la red 192.168.88.0/24).

#### Cliente DHCP
```
/ip dhcp-client
add interface=ROUTER_ONE
```

Permite que el router obtenga configuración dinámica del router externo, incluyendo rutas adicionales si es necesario.

### 6.2 Enrutamiento IPv6

```
/ipv6 route
add blackhole comment="IPv6 solo interno" dst-address=::/0
```

**Política actual**: IPv6 está configurado únicamente para uso interno. Todo el tráfico IPv6 con destino externo es descartado mediante una ruta blackhole.

**Justificación**: El proyecto se centra en implementar una red dual-stack interna con autoconfiguración IPv6, pero sin salida IPv6 a Internet.

---

## 7. NAT y Firewall

### 7.1 NAT (Network Address Translation)

#### Masquerade
```
/ip firewall nat
add action=masquerade chain=srcnat out-interface=ROUTER_ONE \
    src-address=192.168.0.0/16
```

**Propósito**: Todas las redes internas (192.168.0.0/16) se enmascaran con la IP pública o de salida del router cuando acceden a Internet a través de ROUTER_ONE.

### 7.2 Firewall

#### Reglas de Filter

**Regla 1: Aceptar tráfico QUIC (UDP/443)**
```
add action=accept chain=forward dst-address=192.168.50.20 dst-port=443 \
    protocol=udp
```
- Permite tráfico QUIC hacia el reverse proxy en VLAN 50
- Necesario para HTTP/3

**Regla 2: Aceptar conexiones establecidas**
```
add action=accept chain=input comment="01-INPUT: Accept stablished/related" \
    connection-state=established,related
```
- Acepta tráfico de respuesta a conexiones ya establecidas

### 7.3 Address Lists

El router define listas de direcciones para facilitar la gestión del firewall:

```
/ip firewall address-list
add address=192.168.0.0/16 list=internas
add address=192.168.10.0/24 comment="VLAN 10 -Gestion" list=admin
add address=192.168.20.20 comment="DNS Primary" list=dns-servers
add address=192.168.20.21 comment="DNS Secondary" list=dns-servers
add address=192.168.20.2 comment="Kea DHCP" list=dhcp-servers
add address=192.168.20.60 comment="Chrony NTP" list=ntp-servers
add address=192.168.50.10 comment="Caddy Web 1" list=web-servers
add address=192.168.50.11 comment="Caddy Web 2" list=web-servers
add address=192.168.50.20 comment="Reverse Proxy" list=web-servers
add address=192.168.40.50 comment=Dovecot/Postfix list=mail-servers
```

**Servicios**:
- **DNS**: 192.168.20.20 (primario), 192.168.20.21 (secundario)
- **DHCP**: 192.168.20.2 (Kea)
- **NTP**: 192.168.20.60 (Chrony)
- **Web**: 192.168.50.10, 192.168.50.11, 192.168.50.20
- **Mail**: 192.168.40.50 (Dovecot/Postfix)

---

## 8. SNMP (Monitoreo)

El servicio SNMP está **habilitado** para permitir monitoreo remoto del router.

### 8.2 Comunidades SNMP

```
/snmp community
add addresses=192.168.0.0/16 name=gponRouter
add addresses=192.168.173.0/24 name=librenms
add addresses=192.168.173.0/24 name=nms
```

**Comunidades configuradas**:
- **gponRouter**: Accesible desde toda la red interna (192.168.0.0/16)
- **librenms** y **nms**: Específicas para herramientas de monitoreo

---

## 9. Sincronización de Tiempo (NTP)

### 9.1 Cliente NTP

```
/system ntp client
set enabled=yes

/system ntp client servers
add address=192.168.20.60
```

**Servidor NTP interno**: 192.168.20.60 (Chrony en VLAN 20)

El router sincroniza su reloj del sistema con este servidor local, que a su vez se sincroniza con servidores NTP públicos.

### 9.2 Zona Horaria

```
/system clock
set time-zone-name=America/Bogota
```

**Zona horaria**: America/Bogota (UTC-5)

---

## 10. Interfaces y Bridges

### 10.1 Bridge VLAN 20

```
/interface bridge
add name=Bridge-VLAN20

/interface bridge port
add bridge=Bridge-VLAN20 interface=ether14
add bridge=Bridge-VLAN20 interface=vlan20-Core
```

Se creó un bridge que une:
- **ether14** (puerto físico)
- **vlan20-Core** (interfaz virtual VLAN 20)

Esto con el fin de establecer comunicación entre el dispositivo encargado de alojar el servicio de LibreQoS

### 10.2 Interfaz Loopback

```
/interface bridge
add fast-forward=no name=loopback
```

Interfaz loopback creada pero no configurada aún. Puede utilizarse para:
- IP de gestión permanente
- Servicios de routing dinámico
- Terminación de túneles

---

## 12. Servicios del Proyecto

### 12.1 Arquitectura de Servicios

Los servicios están distribuidos en VLANs específicas según su función:

#### VLAN 20 - Core (192.168.20.0/24)
- **DNS Primario**: 192.168.20.20
- **DNS Secundario**: 192.168.20.21
- **DHCP (Kea)**: 192.168.20.2
- **NTP (Chrony)**: 192.168.20.60

#### VLAN 40 - Servicios (192.168.40.0/24)
- **Mail Server**: 192.168.40.50 (Dovecot/Postfix)

#### VLAN 50 - Web (192.168.50.0/24)
- **Caddy Web 1**: 192.168.50.10
- **Caddy Web 2**: 192.168.50.11
- **Reverse Proxy**: 192.168.50.20 

### 12.2 Red GPON

#### VLAN 100 - Clientes GPON
- **Rango IP**: 192.168.100.x
- **Gateway**: 192.168.100.1 (este router)
- **DHCP**: Relay hacia 192.168.20.2
- **Conectividad**: ONTs Huawei EG8145V5 → OLT EA5800-X2 → Switch Cisco → Router

**Rango de clientes DHCP**: 192.168.100.100-192.168.100.200 (según configuración de Kea)

## 13. Diagrama de Flujo de Tráfico

### 13.1 Cliente GPON → Internet

1. ONT (192.168.100.x) → OLT → Switch (VLAN 100) → Router Interno
2. Router verifica firewall (estado de conexión)
3. Router aplica NAT (masquerade)
4. Router envía tráfico a Router Externo (192.168.88.1)
5. Router Externo envía a Internet

### 13.2 Cliente GPON → Servidor Web Interno

1. ONT (192.168.100.x) → Router (VLAN 100)
2. Router enruta hacia VLAN 50 (192.168.50.20)
3. Reverse Proxy procesa solicitud
4. Respuesta regresa por el mismo camino

### 13.3 Solicitud DHCP desde ONT

1. ONT envía DHCPDISCOVER (broadcast en VLAN 100)
2. Router (192.168.100.1) recibe solicitud
3. DHCP Relay reenvía a 192.168.20.2 (unicast)
4. Kea DHCP responde con oferta
5. Router reenvía respuesta a ONT
6. ONT confirma y obtiene IP asignada

---

## 18. Referencias y Recursos

### 18.1 Documentación Oficial

- [MikroTik RouterOS Documentation](https://help.mikrotik.com)
- [CCR2004 Series Manual](https://help.mikrotik.com/docs/display/ROS/CCR2004)
- [VLAN Configuration Guide](https://wiki.mikrotik.com/wiki/Manual:Interface/VLAN)
- [DHCP Relay Setup](https://help.mikrotik.com/docs/display/ROS/DHCP#DHCP-DHCPRelay)
---

