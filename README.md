# Servicios en Red - Práctica 3: DHCP

El protocolo **DHCP (Dynamic Host Configuration Protocol)** es un protocolo de red que permite a los dispositivos obtener configuraciones de red automáticamente, como direcciones IP, máscaras de subred, puertas de enlace predeterminadas y servidores DNS.

Gracias a DHCP no es necesario configurar manualmente cada equipo, lo que simplifica mucho la administración de redes medianas y grandes.

---

## Funcionamiento de DHCP

DHCP opera en un modelo **cliente-servidor**. El servidor DHCP administra un rango de direcciones IP y otros parámetros de configuración de red, mientras que los clientes DHCP solicitan esta información cuando se conectan a la red.

Cuando un dispositivo se conecta a una red, sigue estos pasos para obtener una configuración de red mediante DHCP:

1. **Descubrimiento (DHCPDISCOVER)**  
   El cliente envía un mensaje de *broadcast* buscando un servidor DHCP en la red.

2. **Oferta (DHCPOFFER)**  
   Uno o varios servidores DHCP responden con una oferta que incluye una dirección IP disponible y otros parámetros de red.

3. **Solicitud (DHCPREQUEST)**  
   El cliente elige una de las ofertas (normalmente la primera que recibe) y envía un mensaje DHCPREQUEST solicitando formalmente esa configuración.

4. **Reconocimiento (DHCPACK)**  
   El servidor confirma la asignación con un mensaje DHCPACK. A partir de ese momento la IP y la configuración quedan “reservadas” para el cliente durante un tiempo.

Este ciclo completo se suele resumir como **DORA** (Discover–Offer–Request–Acknowledge).

---

## Configuraciones Gestionadas por DHCP

DHCP puede gestionar varios parámetros de configuración de red, incluyendo:

- **Dirección IP**  
  La dirección única asignada al dispositivo **en la red**.

- **Máscara de subred**  
  Define qué parte de la dirección IP corresponde a la red y qué parte a los hosts.

- **Puerta de enlace predeterminada (gateway)**  
  La dirección IP del router que conecta la red local con otras redes (por ejemplo, con Internet).

- **Servidores DNS**  
  Las direcciones IP de los servidores DNS que los clientes utilizarán para resolver nombres de dominio.

Además, puede proporcionar otros parámetros menos habituales (servidores NTP, dominio de búsqueda, rutas estáticas, etc.), pero para nuestro nivel nos centraremos en los más importantes.

---

## Otros Aspectos de DHCP

- **Concesión de direcciones (lease)**  
  Las direcciones IP asignadas por DHCP tienen un **tiempo de arrendamiento**.  
  Pasado ese tiempo, el cliente:

  - Intenta **renovar** la concesión.
  - Si no puede, la dirección vuelve a estar disponible para otros clientes.

  Esto permite una gestión eficiente del espacio de direcciones IP, especialmente cuando los equipos entran y salen de la red (móviles, portátiles, etc.).

- **Reservas**  
  Los administradores de red pueden **reservar** direcciones IP específicas para ciertos dispositivos basándose en sus direcciones MAC.  
  Ejemplo típico: impresoras de red, servidores internos, dispositivos que queremos que **siempre tengan la misma IP**, pero sin configurarlos manualmente.

- **Exclusiones**  
  Se pueden definir rangos de direcciones IP **que no deben ser asignadas** por el servidor DHCP (por ejemplo, IPs usadas para servidores con IP estática).

---

## Recordatorio de Redes

Antes de continuar con la parte de servidor DHCP en Linux, recordamos algunos conceptos básicos de redes que usaremos constantemente:

### Direcciones IP y rangos

- Una red típica de laboratorio podría ser, por ejemplo: `192.168.1.0/24`.
- Esto significa:
  - **Red**: `192.168.1.0`
  - **Broadcast**: `192.168.1.255`
  - **Hosts disponibles**: de `192.168.1.1` a `192.168.1.254`

En una práctica con DHCP es habitual que:

- El **servidor DHCP** tenga una IP fija (por ejemplo, `192.168.1.1`).
- Los **clientes** obtengan IPs de un rango (por ejemplo, `192.168.1.100 – 192.168.1.200`).

### Máscara de subred

La máscara define el tamaño de la red. Algunos ejemplos:

- `255.255.255.0` → `/24`, 256 direcciones (254 útiles para hosts).
- `255.255.255.128` → `/25`, 128 direcciones (126 útiles).
- `255.255.128.0` → `/17`, 32,768 direcciones (32,766 útiles).
- `255.255.0.0` → `/16`, 65,536 direcciones (65,534 útiles).
  
En las prácticas usaremos redes tipo `/24` por simplicidad.

### Gateway y acceso a otras redes

La **puerta de enlace por defecto** (gateway) es la dirección IP del dispositivo (router) que conecta la red local con otras redes (por ejemplo, Internet).

### DNS

- Son los servidores que traducen nombres (por ejemplo, `google.com`) a direcciones IP.
- El las prácticas usaremos casi siempre servidores DNS públicos como `8.8.8.8` (Google).

---

## Elementos de una Infraestructura DHCP

En una red con DHCP distinguimos:

- **Servidor DHCP**  
  Equipo que **asigna direcciones IP** y otros parámetros de red.  
  En Linux usaremos el servicio **isc-dhcp-server** como servidor.

- **Clientes DHCP**  
  Son los dispositivos que piden una dirección IP (PCs, móviles, VMs, impresoras, etc.).  
  Concretamente, es un proceso o servicio que corre en el dispositivo.
---

## Tipos de Asignación DHCP

Aunque en el día a día solemos hablar simplemente de “DHCP”, internamente podemos distinguir:

1. **Asignación dinámica**  
   - El servidor tiene un **pool** o rango de IPs.
   - Asigna una IP libre a cada cliente mientras dure su lease.
   - Es el modo más habitual en redes de usuarios.

2. **Asignación fija / reserva estática por MAC**  
   - Se configura en el servidor una relación **MAC → IP**.
   - El mismo dispositivo recibe siempre la **misma IP** aunque use DHCP.
   - Muy útil para servidores, impresoras, etc.

3. **Asignación manual (fuera de DHCP)**  
   - Se configura la IP a mano en el equipo.
   - El servidor DHCP no participa.
   - Conviene que esas IPs queden **fuera del rango DHCP** para evitar conflictos.

---

## Servidor DHCP en Linux: isc-dhcp-server

En Debian/Ubuntu, el servidor DHCP más típico es **isc-dhcp-server**.

En esta práctica, una de las máquinas Debian actuará como **servidor DHCP** para la red interna de VirtualBox, y el resto como **clientes**.

### Archivos importantes

- `/etc/dhcp/dhcpd.conf`  
  Archivo principal de configuración del servidor DHCP.

- `/var/lib/dhcp/dhcpd.leases`  
  Archivo donde el servidor guarda los **arrendamientos** (qué IP tiene cada cliente, durante cuánto tiempo, etc.).

- `/etc/default/isc-dhcp-server`  
  Archivo donde se indica **en qué interfaz de red** debe escuchar el servidor DHCP (por ejemplo, `enp0s8`).

---

## Parte práctica