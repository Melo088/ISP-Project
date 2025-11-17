# Proyecto ISP - Infraestructura GPON

## Descripción General
Implementación de una red GPON (Gigabit Passive Optical Network) para entorno académico, integrando dispositivos físicos y servicios virtualizados en Ubuntu Server. El proyecto incluye segmentación de red mediante VLANs, servicios core de red, monitoreo, calidad de servicio (QoS), y provisión de servicios a clientes finales.

## Topología de Red


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

| VLAN | Nombre | Subred | Gateway | Propósito |
|------|--------|--------|---------|-----------|
| 10 | Gestión | 192.168.10.0/24 | 192.168.10.1 | Administración de dispositivos |
| 20 | Servicios_Core | 192.168.20.0/24 | 192.168.20.1 | DHCP, DNS, NTP |
| 30 | Monitoreo | 192.168.30.0/24 | 192.168.30.1 | Prometheus, Grafana, LibreNMS |
| 40 | Email | 192.168.40.0/24 | 192.168.40.1 | Postfix, Dovecot |
| 50 | Web | 192.168.50.0/24 | 192.168.50.1 | Caddy Web y Reverse Proxy |
| 100 | Clientes_GPON | 192.168.100.0/24 | 192.168.100.1 | Clientes finales ONTs |

## Servicios Virtualizados

Todos los servicios están implementados en máquinas virtuales sobre Ubuntu Server 24.04.

### VLAN 20 - Servicios Core

| Servicio | IP | Puertos | Descripción |
|----------|----|---------|-|
| Kea DHCP | 192.168.20.2 | UDP 67-68 (DHCPv4), UDP 546-547 (DHCPv6) | Asignación dinámica de IPs |
| BIND9 DNS | 192.168.20.20, 192.168.20.21 | TCP/UDP 53 | Resolución de nombres |
| Chrony NTP | 192.168.20.60 | UDP 123 | Sincronización de tiempo |

### VLAN 30 - Monitoreo

| Servicio | IP | Puertos | Descripción |
|----------|----|---------|-|
| Prometheus | 192.168.20.40 | TCP 9090 | Recolección de métricas |
| Grafana | 192.168.20.40 | TCP 3000 | Visualización de métricas |
| LibreNMS | 192.168.20.40 | TCP 8000 | Monitoreo SNMP de red |

### VLAN 30 - QoS

| Servicio | IP | Puertos | Descripción |
|----------|----|---------|-|
| LibreQoS | 192.168.20.50 | TCP 9123 | Gestión de calidad de servicio |

### VLAN 40 - Email

| Servicio | IP | Puertos | Descripción |
|----------|----|---------|-|
| Dovecot | 192.168.40.50 | TCP 143/993 (IMAP/S), TCP 110/995 (POP3/S) | Servidor de correo entrante |
| Postfix | 192.168.40.50 | TCP 25 (SMTP), TCP 587 (submission), TCP 465 (SMTPS) | Servidor de correo saliente |

### VLAN 50 - Web

| Servicio | IP | Puertos | Descripción |
|----------|----|---------|-|
| Caddy Web | 192.168.50.10, 192.168.50.11 | TCP 80, TCP 443 | Servidor web |
| Caddy Reverse Proxy | 192.168.50.20 | TCP 80, TCP 443 | Proxy inverso |

## Red GPON - VLAN 100

### Direccionamiento Clientes

- **Rango**: 192.168.100.100 - 192.168.100.200
- **Gateway**: 192.168.100.1
- **DNS**: 192.168.20.20, 192.168.20.21
- **DHCP**: Asignado por Kea (192.168.20.2)


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
│       ├── monitoreo/
│       ├── qos/                     # LibreQoS
│       ├── smtp/
│       └── web/
├── direccionamiento/
├── topologia/
│
└── README.md
```


## Tecnologías Utilizadas

- **Virtualización**: KVM/QEMU en Ubuntu Server
- **Routing**: MikroTik RouterOS
- **Switching**: Cisco SG350X
- **GPON**: Huawei OLT/ONT
- **DHCP**: Kea DHCP Server
- **DNS**: BIND9
- **NTP**: Chrony
- **Web**: Caddy Server
- **Email**: Postfix + Dovecot
- **Monitoreo**: LibreNMS, Prometheus, Grafana
- **QoS**: LibreQoS

---

**Fecha de actualización**: Noviembre 2025  
**Versión**: 2.0
