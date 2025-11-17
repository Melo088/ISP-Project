# Proyecto ISP - Infraestructura GPON

## Descripción General
Implementación de una red GPON (Gigabit Passive Optical Network) para entorno académico, integrando dispositivos físicos y servicios virtualizados en Ubuntu Server. El proyecto incluye segmentación de red mediante VLANs, servicios core de red, monitoreo, calidad de servicio (QoS), y provisión de servicios a clientes finales en una infraestructura con soporte dual-stack IPv4/IPv6.

## Topología de Red

![Imgur Image](https://i.imgur.com/I5o7OSE.jpg)

### Dispositivos Físicos

| Dispositivo | Modelo | Función |
|-------------|--------|---------|
| Router Externo | MikroTik RB3011UiAS-RM | Conexión a Internet |
| Router Interno | MikroTik CCR2004-16G-2S+PC | Core del proyecto, VLAN routing |
| Switch | Cisco SG350X-24 | Distribución con soporte VLAN/trunk |
| OLT | Huawei EA5800-X2 | Cabecera de red GPON |
| ONT | Huawei EchoLife EG8145V5 | Terminal óptico de cliente |
| Splitter | FiberHome Celcia | Divisor óptico pasivo |
| Servidor | Laptop Ubuntu Server 24.04 | Virtualización de servicios |

### Conexiones Físicas

- **Router Externo eth8** → MinisForum Venus LAN2 (QoS)
- **Router Externo** → **Router Interno eth1**
- **Router Interno eth2** → **Switch G1/0/2**
- **Router Interno eth14** → MinisForum Venus LAN1 (QoS)
- **Switch G1/0/1** → **OLT**
- **Switch G1/0/10** → **Servidor**
- **OLT** → **Splitter** → **ONTs** (fibra óptica)

## Segmentación de Red - VLANs

| VLAN | Nombre | Subred IPv4 | Subred IPv6 | Gateway IPv4 | Gateway IPv6 | Propósito |
|------|--------|-------------|-------------|--------------|--------------|-----------|
| 10 | Gestión | 192.168.10.0/24 | 2001:db8:10::/64 | 192.168.10.1 | 2001:db8:10::1 | Administración de dispositivos |
| 20 | Servicios_Core | 192.168.20.0/24 | 2001:db8:20::/64 | 192.168.20.1 | 2001:db8:20::1 | DHCP, DNS, NTP, Prometheus, Grafana, LibreNMS  |
| 40 | Email | 192.168.40.0/24 | 2001:db8:40::/64 | 192.168.40.1 | 2001:db8:40::1 | Postfix, Dovecot |
| 50 | Web | 192.168.50.0/24 | 2001:db8:50::/64 | 192.168.50.1 | 2001:db8:50::1 | Caddy Web y Reverse Proxy |
| 100 | Clientes_GPON | 192.168.100.0/24 | 2001:db8:100::/64 | 192.168.100.1 | 2001:db8:100::1 | Clientes finales ONTs |

## Servicios Virtualizados

Los servicios están implementados en máquinas virtuales sobre Ubuntu Server 24.04 con direccionamiento dual-stack (IPv4/IPv6).

![Imgur Image](https://i.imgur.com/iBXDZ4Q.jpg)

### VLAN 20 - Servicios Core

#### Kea DHCP
Desplegamos Kea DHCP como servidor de asignación automática de direcciones IP para los clientes de la VLAN 100. 

#### BIND9 DNS
El servicio de resolución de nombres está implementado con BIND9, utilizando dos servidores (primario y secundario).

#### Chrony NTP
Para la sincronización horaria de todos los dispositivos de red implementamos Chrony como servidor NTP. La sincronización precisa del tiempo es fundamental para el correcto funcionamiento de logs, autenticación y correlación de eventos entre los diferentes sistemas que componen la infraestructura.

#### Prometheus
Prometheus actúa como sistema central de recolección de métricas de todos los servicios y dispositivos. Mediante exporters especializados, captura información de rendimiento, disponibilidad y salud de la infraestructura, almacenándola en su base de datos de series temporales para análisis histórico y detección de anomalías.

#### Grafana
Sobre las métricas recolectadas por Prometheus, Grafana proporciona dashboards interactivos que permiten visualizar el estado de la red en tiempo real. Los paneles personalizados facilitan la identificación rápida de problemas y el análisis de tendencias, mejorando significativamente la capacidad de respuesta del equipo operativo.

#### LibreNMS
Implementamos LibreNMS para el monitoreo SNMP de los dispositivos de red físicos. Este sistema proporciona mapas de topología, alertas de caídas de enlaces y seguimiento de ancho de banda, etc.

#### LibreQoS
Para garantizar calidad de servicio diferenciada, desplegamos LibreQoS como sistema de gestión de ancho de banda y priorización de tráfico. Integrado con los routers MikroTik mediante interfaces dedicadas (MinisForum Venus), permite aplicar políticas de QoS por cliente GPON, asegurando experiencia óptima para servicios críticos.

### VLAN 40 - Email

| Servicio | IPv4 | IPv6 | Puertos | Descripción |
|----------|------|------|---------|-------------|
| Dovecot | 192.168.40.50 | 2001:db8:40::50 | TCP 143/993 (IMAP/S), TCP 110/995 (POP3/S) | Servidor de correo entrante |
| Postfix | 192.168.40.50 | 2001:db8:40::50 | TCP 25 (SMTP), TCP 587 (submission), TCP 465 (SMTPS) | Servidor de correo saliente |

#### Postfix
Postfix funciona como MTA (Mail Transfer Agent) encargado del envío y enrutamiento de correo electrónico saliente. Configurado con soporte TLS/SSL, maneja el relay SMTP seguro y la entrega de mensajes, integrándose con sistemas de autenticación para prevenir uso no autorizado del servidor.

#### Dovecot
Dovecot complementa a Postfix proporcionando servicios de recuperación de correo mediante protocolos IMAP y POP3, ambos con sus variantes seguras. Los usuarios pueden acceder a sus buzones de manera segura desde diferentes clientes, con soporte para múltiples carpetas y sincronización eficiente de mensajes.

### VLAN 50 - Web

#### Caddy Web Servers
Implementamos dos instancias de Caddy como servidores web para alojar aplicaciones y contenido del proyecto. Caddy ofrece configuración automática de certificados HTTPS mediante Let's Encrypt.

#### Caddy Reverse Proxy
El proxy inverso Caddy actúa como punto de entrada único para los servicios web, distribuyendo el tráfico entre los dos servidores web backend. Esta configuración proporciona balanceo de carga básico y permite realizar mantenimiento en servidores individuales sin interrumpir el servicio, mejorando la disponibilidad general.

## Red GPON - VLAN 100

### Direccionamiento Clientes

#### IPv4
- **Rango**: 192.168.100.100 - 192.168.100.200
- **Gateway**: 192.168.100.1
- **DNS**: 192.168.20.20, 192.168.20.21
- **DHCP**: Asignado por Kea (192.168.20.2)

### Arquitectura GPON

La red GPON constituye el segmento de acceso del proyecto, conectando clientes finales mediante tecnología de fibra óptica pasiva. Los ONTs Huawei EG8145V5 se conectan a la OLT EA5800-X2 . Los clientes en VLAN 100 obtienen direccionamiento automático y acceso a todos los servicios de red mediante enrutamiento entre VLANs configurado en el router interno MikroTik.

## Tecnologías Utilizadas

- **Virtualización**: en Ubuntu Server
- **Routing**: MikroTik RouterOS
- **Switching**: Cisco SG350X
- **GPON**: Huawei OLT/ONT
- **DHCP**: Kea DHCP Server (DHCPv4)
- **DNS**: BIND9
- **NTP**: Chrony
- **Web**: Caddy Server
- **Email**: Postfix + Dovecot
- **Monitoreo**: LibreNMS, Prometheus, Grafana
- **QoS**: LibreQoS

## Estructura del Repositorio

```
proyecto-gpon-infra2/
├── dispositivos/
│   ├── olt/
│   │
│   ├── switch/
│   │
│   └── router/
│
├── servidor/
│   └── vms/
│       ├── dhcp/                    # Kea DHCP
│       ├── dns/                     # BIND9
│       ├── ntp/                     # Chrony
│       ├── monitoreo/               # Grafana & Prometheus, LibreNMS
│       ├── qos/                     # LibreQoS
│       ├── smtp/                    # Postfix, Dovecot
│       └── web/                     # Caddy
├── direccionamiento/
├── topologia/
│
└── README.md
```

