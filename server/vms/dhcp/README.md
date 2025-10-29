# Infraestructura Red GPON - SERVICIO DHCP KEA
## 1. INTRODUCCIÓN AL SERVICIO DHCP

El servicio DHCP (Dynamic Host Configuration Protocol) implementado mediante Kea DHCP Server permite la asignación automática de direcciones IP y parámetros de configuración de red a los clientes ONT conectados a la red GPON. Este servicio opera en la red 192.168.100.0/24 asignada a la VLAN 100 para clientes GPON.

## 2. PROCESO DHCP - SECUENCIA DORA

El protocolo DHCP utiliza un proceso de cuatro fases conocido como DORA (Discovery, Offer, Request, Acknowledgement) para asignar direcciones IP a los clientes. A continuación se presenta un diagrama que describe dicha secuencia:
![Imgur Image](https://i.imgur.com/Ra6OSvS.jpg)

**Puertos UDP Utilizados**
El servicio DHCP opera mediante el protocolo UDP en dos puertos específicos:​
- Puerto 67: Escucha del servidor DHCP (Kea)
- Puerto 68: Escucha del cliente DHCP (ONTs)

## 3. ARQUITECTURA DEL SERVICIO KEA DHCP

![Imgur Image](https://i.imgur.com/8qE04CY.jpg)

## 4. CONFIGURACIÓN DEL SERVIDOR KEA DHCP

### 4.1 Configuración de Interfaces
El servidor Kea DHCP está configurado para escuchar en la interfaz `eth1` mediante el parámetro `interfaces-config`. Esta configuración incluye parámetros de socket y reintentos:

```yaml
"interfaces-config": {
    "interfaces": ["eth1"],
    "dhcp-socket-type": "udp",
    "service-sockets-max-retries": 5000,
    "service-sockets-retry-wait-time": 5000
}
```

**Parámetros:**
- `interfaces`: Define la interfaz de red que escuchará solicitudes DHCP (eth1)
- `dhcp-socket-type`: Tipo de socket UDP en lugar de raw sockets para optimización
- `service-sockets-max-retries`: Número máximo de reintentos para abrir sockets (5000)
- `service-sockets-retry-wait-time`: Tiempo de espera entre reintentos en milisegundos (5000)

### 4.2 Temporizadores de Arrendamiento (Lease Timers)
Los temporizadores controlan el ciclo de vida de las direcciones IP asignadas. La configuración implementada define tres valores temporales críticos:

```yaml
"valid-lifetime": 28800,
"renew-timer": 14400,
"rebind-timer": 25200
```

```yaml
gantt
    title Ciclo de Vida del Lease DHCP (8 horas)
    dateFormat X
    axisFormat %H:%M
    
    section Lease Activo
    Uso Normal (0-4h)          :active, 0, 14400000
    
    section Renovación T1
    Estado RENEWING (4-7h)     :crit, 14400000, 10800000
    
    section Rebinding T2
    Estado REBINDING (7-8h)    :done, 25200000, 3600000
    
    section Expiración
    Lease Expira (8h)          :milestone, 28800000, 0
```
- **valid-lifetime (28800s = 8 horas)**: Tiempo total de validez del arrendamiento IP. Después de este periodo, el lease expira completamente
- **renew-timer (14400s = 4 horas)**: Temporizador T1 al 50% del lease. El cliente entra en estado RENEWING y contacta al servidor original mediante unicast para extender el lease
- **rebind-timer (25200s = 7 horas)**: Temporizador T2 al 87.5% del lease. Si no recibió respuesta en T1, el cliente entra en estado REBINDING y envía broadcast buscando cualquier servidor DHCP


### 4.3 Base de Datos de Arrendamientos
La configuración de lease-database define el almacenamiento persistente de las asignaciones IP:

```yaml
"lease-database": {
    "type": "memfile",
    "persist": true,
    "name": "/var/lib/kea/dhcp4.csv",
    "lfc-interval": 3600
}
```

**Parámetros de almacenamiento:**

- `type`: "memfile" indica almacenamiento en archivo plano optimizado para rendimiento​​
- `persist`: true garantiza escritura persistente de leases en disco​
- `name`: Ruta del archivo CSV donde se registran los arrendamientos​
- `lfc-interval`: Intervalo de limpieza de archivo (3600s = 1 hora) para optimizar espacio​​

**Estructura del archivo de leases:**
El archivo `/var/lib/kea/dhcp4.csv` almacena los registros con el siguiente formato:

```csv
address,hwaddr,client_id,valid_lifetime,expire,subnet_id,fqdn_fwd,fqdn_rev,hostname,state,user_context
192.168.100.100,aa:bb:cc:dd:ee:ff,01:aa:bb:cc:dd:ee:ff,28800,1730212800,1,0,0,ont-cliente1,0,
```

### 4.4 Socket de Control
El socket de control Unix permite la administración en tiempo real del servidor sin reiniciarlo:

```yaml
"control-socket": {
    "socket-type": "unix",
    "socket-name": "/tmp/kea4-ctrl-socket"
}
```
Este socket permite ejecutar comandos administrativos mediante kea-shell para consultar estadísticas, modificar configuraciones y gestionar leases dinámicamente.

## 5. CONFIGURACIÓN DE SUBRED Y POOL DE DIRECCIONES

### 5.1 Definición de Subred
La subred 192.168.100.0/24 está asignada a la VLAN 100 para clientes GPON conectados a través de la OLT:

```yaml
"subnet4": [
    {
        "subnet": "192.168.100.0/24",
        "id": 1,
        "pools": [ 
            { "pool": "192.168.100.100 - 192.168.100.200" } 
        ]
    }
]
```

**Distribución de direcciones:**
- Rango total: 192.168.100.0 - 192.168.100.255 (256 direcciones)​
- Pool DHCP: 192.168.100.100 - 192.168.100.200 (101 direcciones asignables)​
- Gateway: 192.168.100.1 (Router interno)

![Imgur Image](https://i.imgur.com/FnZwi40.jpg)

### 5.2 Opciones DHCP Configuradas

```yaml
"option-data": [
    {
        "name": "routers",
        "data": "192.168.100.1"
    },
    {
        "name": "domain-name-servers",
        "data": "8.8.8.8, 8.8.4.4"
    },
    {
        "name": "domain-search",
        "data": "plataformas.tel"
    }
]
```

- `routers`: Define la puerta de enlace predeterminada 192.168.100.1. Los paquetes hacia redes externas se enrutan a través del router interno​​
- `domain-name-servers`: Servidores DNS públicos de Google (8.8.8.8 primario, 8.8.4.4 secundario). Proporcionan resolución de nombres de dominio para los clientes ONT​​
- `domain-search`: Sufijo de búsqueda DNS "plataformas.tel". Permite resolución de nombres cortos dentro del dominio local

## 6. CONFIGURACIÓN DE LOGGING Y MONITOREO
### 6.1. Sistema de Logs

El servidor Kea DHCP está configurado con logging detallado para depuración y auditoría:

```yaml
"loggers": [
    {
        "name": "kea-dhcp4",
        "output_options": [
            {
                "output": "/var/log/kea-dhcp4.log"
            }
        ],
        "severity": "DEBUG",
        "debuglevel": 99
    }
]
```

**Niveles de severidad disponibles:**
- FATAL: Errores críticos que detienen el servicio
- ERROR: Errores operacionales que afectan funcionalidad
- WARN: Advertencias de situaciones anormales
- INFO: Información operacional general
- DEBUG: Información detallada para diagnóstico (nivel actual)

**Debug level 99:** Máxima verbosidad, registra cada paquete DHCP, transición de estado y operación de base de datos. Útil para depuración pero genera gran volumen de logs.

## 6.2. Monitoreo de Arrendamientos

![Imgur Image](https://i.imgur.com/Dy61zDR.jpg)


## 7. CONFIGURACIÓN DE RED DEL SERVIDOR

### 7.1. Configuración Netplan

La interfaz eth1 del servidor está configurada mediante Netplan con dirección estática:

```yaml
network:
  version: 2
  ethernets:
    eth1:
      addresses: [192.168.20.2/24]
      routes:
        - to: default
          via: 192.168.20.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### 7.2 Integración con la Infraestructura

![Imgur Image](https://i.imgur.com/bovebBw.jpg)

1. ONT conectado a OLT envía DHCPDISCOVER en VLAN 100​
2. Switch reenvía el broadcast al puerto G1/0/10 (Servidor)​
3. Servidor Kea DHCP recibe solicitud en eth1​
4. Servidor asigna IP del pool 192.168.100.100-200​
5. Respuesta DHCPOFFER/DHCPACK regresa por el mismo camino​
6. ONT configura IP, gateway y DNS recibidos

## 8. COMANDOS DE ADMINISTRACIÓN Y VERIFICACIÓN

### 8.1. Gestión del Servicio

```bash
# Verificar configuración antes de aplicar
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf

# Iniciar servicio
sudo systemctl start kea-dhcp4-server

# Detener servicio
sudo systemctl stop kea-dhcp4-server

# Reiniciar servicio
sudo systemctl restart kea-dhcp4-server

# Estado del servicio
sudo systemctl status kea-dhcp4-server

# Habilitar inicio automático
sudo systemctl enable kea-dhcp4-server
```

### 8.2. Monitoreo y Auditoría
```bash
# Ver leases activos
cat /var/lib/kea/dhcp4.csv

# Monitoreo en tiempo real de logs
tail -f /var/log/kea-dhcp4.log

# Buscar leases de MAC específica
grep "aa:bb:cc:dd:ee:ff" /var/lib/kea/dhcp4.csv

# Contar leases activos
wc -l /var/lib/kea/dhcp4.csv

# Ver estadísticas del pool
grep "192.168.100" /var/lib/kea/dhcp4.csv | wc -l
```

### 8.3. Validaciones de Conectividad

```bash
# Verificar interfaz eth1
ip addr show eth1

# Verificar ruta predeterminada
ip route show

# Verificar puerto UDP 67 escuchando
sudo netstat -ulnp | grep :67

# Test de conectividad al gateway
ping -c 4 192.168.20.1

# Verificar resolución DNS
nslookup google.com 8.8.8.8
```

## 9. FLUJO COMPLETO DE ASIGNACIÓN DHCP

![Imgur Image](https://i.imgur.com/Zqyv3Xt.jpg)



