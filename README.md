# Reto 4 — Grupo 4 (`grupo4reto4.eus`)

Documentación técnica del despliegue de infraestructura, seguridad perimetral y servicios en red sobre Proxmox VE, pfSense y Ubuntu Server.

---

## Tabla de Direccionamiento IP y Servicios

### 1. Subredes y Equipos

| Equipo / Hostname | Alias DNS (`CNAME`) | Interfaz / Red | Dirección IP | Rol y Servicios |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense (`gateway`)** | `pfsense` | WAN (Aula) / LAN (Proxmox) | `172.26.46.54` / `192.168.4.1` | Firewall perimetral, NAT y enrutamiento |
| **`server1-g4`** | `dns1` | LAN (`192.168.4.0/24`) | `192.168.4.101` | DNS Maestro (BIND9), DHCP Primario (Failover) |
| **`server2-g4`** | `dns2` | LAN (`192.168.4.0/24`) | `192.168.4.102` | DNS Esclavo (BIND9), DHCP Secundario (Failover) |
| **`server3-g4`** | `monitor`, `web-proxmox` | LAN (`192.168.4.0/24`) | `192.168.4.103` | Servidor Web Apache2 (`:80`), Monitorix (`:8080`) |
| **`server4-g4`** | `cliente` | LAN (`192.168.4.0/24`) | `192.168.4.104` *(Reserva MAC)* | Cliente Ubuntu de pruebas (`bc:24:11:dc:86:25`) |
| **Pool Dinámico DHCP** | — | LAN (`192.168.4.0/24`) | `192.168.4.150` – `.199` | Rango dinámico gestionado en alta disponibilidad |
| **`web-vmware`** | `www`, `web-server`, `@` | Red VMware | `172.16.250.104` | Servidor Web principal del grupo |
| **`web-interna`** | — | Red Interna | `10.0.4.2` | Servicio web interno |
| **`pc-win1` .. `pc-win5`** | — | Red Aula (`172.26.44.0/24`) | `172.26.44.9` – `.13` | Equipos físicos de administración y pruebas |

### 2. Reglas de Redirección de Puertos (NAT Port Forward en `172.26.46.54`)

| Servicio | Protocolo | Origen Permitido | Puerto Externo (WAN) | IP Interna Destino | Puerto Interno |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SSH `server1-g4`** | TCP | `172.26.44.0/24` | `2241` | `192.168.4.101` | `22` |
| **SSH `server2-g4`** | TCP | `172.26.44.0/24` | `2242` | `192.168.4.102` | `22` |
| **SSH `server3-g4`** | TCP | `172.26.44.0/24` | `2243` | `192.168.4.103` | `22` |
| **SSH `server4-g4`** | TCP | `172.26.44.0/24` | `2244` | `192.168.4.104` | `22` |
| **Web Apache2** | TCP | `172.26.44.0/24` | `8480` | `192.168.4.103` | `80` |
| **Monitorix** | TCP | `172.26.44.0/24` | `8481` | `192.168.4.103` | `8080` |
| **DNS (`grupo4reto4.eus`)** | TCP/UDP | `172.26.44.0/24` | `53` | `192.168.4.101` | `53` |

---

## Diagrama de Topología de Red

```mermaid
flowchart TB
    subgraph AULA["Red Aula / Exterior (172.26.44.0/24 - 172.26.46.0/24)"]
        PC["PCs Windows Aula<br/>172.26.44.9 - 172.26.44.13"]
        VMW["Servidor Web VMware (www)<br/>172.16.250.104"]
    end

    subgraph PFS["Firewall Perimetral pfSense"]
        FW["pfSense<br/>WAN: 172.26.46.54<br/>LAN: 192.168.4.1<br/>(NAT: 2241-2244, 8480, 8481, 53)"]
    end

    subgraph LAN["Red Interna Proxmox - Grupo 4 (192.168.4.0/24)"]
        S1["server1-g4 (dns1)<br/>192.168.4.101<br/>DNS Maestro + DHCP Primary"]
        S2["server2-g4 (dns2)<br/>192.168.4.102<br/>DNS Esclavo + DHCP Secondary"]
        S3["server3-g4 (monitor / web-proxmox)<br/>192.168.4.103<br/>Apache2 (:80) + Monitorix (:8080)"]
        S4["server4-g4 (cliente)<br/>192.168.4.104 (Reserva MAC)<br/>Cliente Ubuntu"]
    end

    PC -- "SSH (2241-2244) / HTTP (8480, 8481) / DNS (53)" --> FW
    PC -- "HTTPS (www.grupo4reto4.eus)" --> VMW
    FW --- S1
    FW --- S2
    FW --- S3
    FW --- S4
    S1 <-- "Transferencia Zona (AXFR) & DHCP Failover" --> S2
```

---

## Fase 0: Preparación Inicial y Acceso Remoto (SSH)

### 0.1. Configuración de Red, Hostnames y Zona Horaria

* Configuramos el direccionamiento IP estático en los servidores de la red interna (`192.168.4.0/24`) con puerta de enlace `192.168.4.1`.
  ![Configuración de red 1](informe/capturas/fase0/confred/image1.png)
  ![Configuración de red 2](informe/capturas/fase0/confred/image2.png)
  ![Configuración de red 3](informe/capturas/fase0/confred/image3.png)
  ![Configuración de red 4](informe/capturas/fase0/confred/image4.png)
  ![Configuración de red 5](informe/capturas/fase0/confred/image5.png)
* Asignamos los nombres de equipo (`server1-g4`, `server2-g4`, `server3-g4`, `server4-g4`) para identificar cada nodo de la infraestructura.
  ![Configuración de hostnames](informe/capturas/fase0/hostnames/hostnames.png)
* Ajustamos la zona horaria (`Europe/Madrid`) en todos los servidores para sincronizar los registros y marcas de tiempo de los servicios.
  ![Zona horaria 1](informe/capturas/fase0/zona%20horaria/image1.png)
  ![Zona horaria 2](informe/capturas/fase0/zona%20horaria/image2.png)
  ![Zona horaria 3](informe/capturas/fase0/zona%20horaria/image3.png)
  ![Zona horaria 4](informe/capturas/fase0/zona%20horaria/image4.png)

### 0.2. Servicio SSH y Autenticación por Clave Pública

* Instalamos `openssh-server`, editamos `/etc/ssh/sshd_config` para habilitar `PubkeyAuthentication yes` y `PasswordAuthentication yes`, y reiniciamos el servicio (`sudo systemctl restart ssh`).
  ![Instalación SSH](informe/capturas/fase0/ssh/image1.png)
  ![Editar sshd_config](informe/capturas/fase0/ssh/image2.png)
  ![PubkeyAuthentication yes](informe/capturas/fase0/ssh/image3.png)
  ![PasswordAuthentication yes](informe/capturas/fase0/ssh/image4.png)
  ![Reinicio servicio SSH](informe/capturas/fase0/ssh/image5.png)
* Habilitamos el servicio SSH en pfSense desde su consola mediante la opción `14`.
  ![SSHD en pfSense](informe/capturas/fase0/ssh/image6.png)
* Generamos el par de claves `ed25519` en nuestro PC Windows (`ssh-keygen -t ed25519 -f D:\ssh\id_ed25519`) sin frase de paso.
  ![Generación clave pública Windows](informe/capturas/fase0/ssh/image7.png)
* Accedemos a la interfaz web de pfSense (`172.26.46.54`) y creamos en **Firewall > NAT** las redirecciones desde la red del aula (`172.26.44.0/24`) hacia cada servidor (`2241`–`2244`).
  ![Login pfSense](informe/capturas/fase0/ssh/image8.png)
  ![Dashboard pfSense](informe/capturas/fase0/ssh/image9.png)
  ![Regla NAT SSH](informe/capturas/fase0/ssh/image10.png)
* Habilitamos `PasswordAuthentication yes` en `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` para copiar nuestra clave pública desde PowerShell y verificamos el acceso mediante clave privada (`ssh -i D:\ssh\id_ed25519 -p 2244 alumno@172.26.46.54`).
  ![Permitir password en cloudimg](informe/capturas/fase0/ssh/image11.png)
  ![Copia de clave pública](informe/capturas/fase0/ssh/image12.png)
  ![Acceso por clave pública](informe/capturas/fase0/ssh/image13.png)

---

## Fase 1: Cortafuegos Perimetral pfSense y Reglas NAT

### 1.1. Desactivar DHCP en pfSense

* Antes de desplegar nuestros propios servidores DHCP, deshabilitamos el servicio DHCP integrado en pfSense desde **Services > DHCP Server > LAN**, desmarcando *Enable DHCP server on LAN interface* y guardando los cambios.
  ![Desactivar DHCP en pfSense](informe/capturas/fase1/cambiar%20dhcp/image1.png)

### 1.2. Creación de Reglas de Port Forwarding (NAT)

* Dado que pfSense aplica por defecto una política restrictiva que bloquea el tráfico entrante desde el exterior, creamos en **Firewall > NAT > Port Forward** las reglas necesarias restringiendo el origen a la red del aula (`172.26.44.0/24`):
  * **SSH (TCP):** Puertos `2241` -> `192.168.4.101:22`, `2242` -> `192.168.4.102:22`, `2243` -> `192.168.4.103:22` y `2244` -> `192.168.4.104:22`.
    ![Regla NAT SSH server1](informe/capturas/fase1/creacion%20reglas/image1.png)
    ![Regla NAT SSH server2](informe/capturas/fase1/creacion%20reglas/image2.png)
    ![Regla NAT SSH server3](informe/capturas/fase1/creacion%20reglas/image3.png)
    ![Regla NAT SSH server4](informe/capturas/fase1/creacion%20reglas/image4.png)
  * **Apache2 (TCP):** Puerto `8480` -> `192.168.4.103:80`.
    ![Regla NAT Apache](informe/capturas/fase1/creacion%20reglas/image5.png)
  * **Monitorix (TCP):** Puerto `8481` -> `192.168.4.103:8080`.
    ![Regla NAT Monitorix](informe/capturas/fase1/creacion%20reglas/image6.png)
  * **DNS (TCP/UDP):** Puerto `53` -> `192.168.4.101:53`.
    ![Regla NAT DNS](informe/capturas/fase1/creacion%20reglas/image7.png)
* Aplicamos la configuración con **Apply Changes** y comprobamos la tabla completa de reglas NAT activas.
  ![Tabla completa reglas NAT](informe/capturas/fase1/creacion%20reglas/image8.png)
  ![Confirmación cambios aplicados](informe/capturas/fase1/creacion%20reglas/image9.png)

---

## Fase 2: Servicio DHCP en Alta Disponibilidad (Failover)

* Instalamos `isc-dhcp-server` en `server1-g4`, identificamos la interfaz `eth0` con `ip a` y la configuramos en `/etc/default/isc-dhcp-server`.
  ![Instalación DHCP server1](informe/capturas/fase2/dhcp/image1.png)
  ![Interfaz eth0](informe/capturas/fase2/dhcp/image2.png)
  ![INTERFACESv4 eth0](informe/capturas/fase2/dhcp/image3.png)
* Configuramos `/etc/dhcp/dhcpd.conf` en `server1-g4` como `primary` (`192.168.4.101`, puerto `647`), repartiendo el rango `192.168.4.150`–`192.168.4.199` y entregando el dominio `grupo4reto4.eus` junto con nuestros DNS (`.101` y `.102`). Validamos la sintaxis (`dhcpd -t`), arrancamos el servicio y comprobamos el estado `Both servers normal` en los logs.
  ![Configuración dhcpd.conf primario](informe/capturas/fase2/dhcp/image4.png)
  ![Validación sintaxis primario](informe/capturas/fase2/dhcp/image5.png)
  ![Estado servicio DHCP primario](informe/capturas/fase2/dhcp/image6.png)
  ![Logs failover Both servers normal](informe/capturas/fase2/dhcp/image7.png)
* Repetimos el despliegue en `server2-g4` como `secondary` (`192.168.4.102`, puerto `847`), validando la sintaxis y el arranque del servicio.
  ![Instalación DHCP server2](informe/capturas/fase2/dhcp/image8.png)
  ![INTERFACESv4 server2](informe/capturas/fase2/dhcp/image9.png)
  ![Configuración dhcpd.conf secundario](informe/capturas/fase2/dhcp/image10.png)
  ![Validación sintaxis secundario](informe/capturas/fase2/dhcp/image11.png)
  ![Estado servicio DHCP secundario](informe/capturas/fase2/dhcp/image12.png)
* Añadimos la reserva por dirección MAC para `server4-g4` (`192.168.4.104`) en ambos servidores, configuramos `dhcp4: true` en su Netplan y comprobamos que obtiene la IP reservada y la mantiene incluso tras detener el servicio DHCP en `server1-g4`.
  ![Reserva MAC server4-g4](informe/capturas/fase2/dhcp/image13.png)
  ![Netplan DHCP en server4-g4](informe/capturas/fase2/dhcp/image14.png)
  ![Obtención de IP en server4-g4](informe/capturas/fase2/dhcp/image15.png)
  ![Parada DHCP en server1-g4](informe/capturas/fase2/dhcp/image16.png)
  ![Persistencia de IP tras caída](informe/capturas/fase2/dhcp/image17.png)

---

## Fase 3: Servicio DNS Maestro-Esclavo (`grupo4reto4.eus`)

* Instalamos `bind9`, `bind9utils` y `dnsutils` en ambos servidores DNS y configuramos `/etc/bind/named.conf.options` con los reenviadores `1.1.1.1` y `8.8.8.8`, validación DNSSEC y escucha exclusiva en IPv4.
  ![Instalación BIND9](informe/capturas/fase3/dns/image1.png)
  ![Configuración named.conf.options](informe/capturas/fase3/dns/image2.png)
* Definimos en `/etc/bind/named.conf.local` del maestro las zonas directa (`grupo4reto4.eus`) e inversa (`4.168.192.in-addr.arpa`) autorizando la transferencia y notificación hacia `192.168.4.102`. Creamos `db.grupo4reto4.eus` (registros `A` y alias `CNAME`: `dns1`, `dns2`, `monitor`, `web-proxmox`, `cliente`, `www`) y `db.192.168.4` (registros `PTR`), validándolos con `named-checkconf` y `named-checkzone`.
  ![named.conf.local maestro](informe/capturas/fase3/dns/image3.png)
  ![Zona directa db.grupo4reto4.eus](informe/capturas/fase3/dns/image4.png)
  ![Zona inversa db.192.168.4](informe/capturas/fase3/dns/image5.png)
  ![Comprobación sintaxis DNS](informe/capturas/fase3/dns/image6.png)
* Configuramos `server2-g4` como esclavo (`type slave`) y verificamos la transferencia automática de las zonas en `/var/cache/bind/`. Asignamos ambos DNS (`192.168.4.101` y `192.168.4.102`) y el dominio `grupo4reto4.eus` en Cloud-Init desde Proxmox.
  ![named.conf.local esclavo](informe/capturas/fase3/dns/image7.png)
  ![Archivos transferidos en esclavo](informe/capturas/fase3/dns/image8.png)
  ![Configuración DNS en Cloud-Init](informe/capturas/fase3/dns/image9.png)
* **Pruebas de funcionamiento, sincronización y alta disponibilidad:**
  * Incrementamos el Serial en el maestro a `2026111603`, recargamos el servicio (`sudo systemctl reload bind9`) y verificamos en `server2-g4` tanto el registro `SOA` actualizado como los logs de transferencia (`Transfer status: success`).
    ![Cambio de serial a 2026111603](informe/capturas/fase3/dns/image10.png)
    ![Reload bind9](informe/capturas/fase3/dns/image11.png)
    ![Verificación SOA en esclavo](informe/capturas/fase3/dns/image12.png)
    ![Logs de transferencia en esclavo](informe/capturas/fase3/dns/image13.png)
  * Comprobamos la resolución externa desde un PC Windows del aula hacia la IP WAN de pfSense (`nslookup www.grupo4reto4.eus 172.26.46.54`).
    ![Consulta DNS desde WAN aula](informe/capturas/fase3/dns/image14.png)
  * Verificamos resoluciones inversas (`dig -x`), directas (`pc-win3`, `monitor`) y acceso web por nombre (`curl server3-g4`) en ambos servidores, comprobando finalmente la continuidad del servicio al detener `bind9` en `server1-g4`.
    ![Resolución inversa maestro](informe/capturas/fase3/dns/image15.png)
    ![Resolución inversa esclavo](informe/capturas/fase3/dns/image16.png)
    ![Resolución directa y CNAME maestro](informe/capturas/fase3/dns/image17.png)
    ![Resolución directa y CNAME esclavo](informe/capturas/fase3/dns/image18.png)
    ![Curl por nombre](informe/capturas/fase3/dns/image19.png)
    ![Parada del DNS maestro](informe/capturas/fase3/dns/image20.png)
    ![Resolución con maestro caído](informe/capturas/fase3/dns/image21.png)
    ![Comprobación adicional DNS 1](informe/capturas/fase3/dns/image22.png)
    ![Comprobación adicional DNS 2](informe/capturas/fase3/dns/image23.png)

---

## Fase 4: Servidor Web Apache y Monitorización con Monitorix

### 4.1. Servidor Web Apache2 (`server3-g4`)

* Instalamos `apache2`, verificamos que el servicio esté activo y creamos `/var/www/grupo4reto4/index.html`.
  ![Instalación Apache2](informe/capturas/fase4/apache/image1.png)
  ![Estado servicio Apache2](informe/capturas/fase4/apache/image2.png)
  ![Creación web grupo4reto4](informe/capturas/fase4/apache/image3.png)
* Creamos el VirtualHost `/etc/apache2/sites-available/grupo4reto4.conf` con logs personalizados (`grupo4reto4_access.log` y `grupo4reto4_error.log`), lo activamos con `a2ensite`, deshabilitamos `000-default.conf` y validamos la configuración con `apache2ctl configtest` (`Syntax OK`) y `apache2ctl -S`.
  ![VirtualHost grupo4reto4.conf](informe/capturas/fase4/apache/image4.png)
  ![Habilitar sitio](informe/capturas/fase4/apache/image5.png)
  ![Configtest Syntax OK](informe/capturas/fase4/apache/image6.png)
  ![Deshabilitar sitio por defecto](informe/capturas/fase4/apache/image7.png)
  ![Apache2ctl -S](informe/capturas/fase4/apache/image8.png)
* Comprobamos el acceso interno por IP (`curl http://192.168.4.103`) y por nombre (`curl http://web-proxmox.grupo4reto4.eus`), el acceso externo desde el navegador del aula (`http://172.26.46.54:8480`) y los registros de visita en `/var/log/apache2/grupo4reto4_access.log`.
  ![Curl interno por IP](informe/capturas/fase4/apache/image9.png)
  ![Acceso desde navegador aula puerto 8480](informe/capturas/fase4/apache/image10.png)
  ![Curl interno por nombre](informe/capturas/fase4/apache/image11.png)
  ![Revisión configuración VirtualHost](informe/capturas/fase4/apache/image12.png)
  ![Ficheros de log de Apache](informe/capturas/fase4/apache/image13.png)
  ![Registros de acceso](informe/capturas/fase4/apache/image14.png)

### 4.2. Monitorización con Monitorix (`server3-g4`)

* Instalamos `monitorix` y `apache2-utils`, habilitamos el servicio (`sudo systemctl enable --now monitorix`) y comprobamos que responde `HTTP/1.0 200 OK` en el puerto `8080`.
  ![Instalación Monitorix](informe/capturas/fase4/monitorix/image1.png)
  ![Estado servicio Monitorix](informe/capturas/fase4/monitorix/image2.png)
  ![Curl local Monitorix](informe/capturas/fase4/monitorix/image3.png)
* Activamos la gráfica de Apache (`apache = y`) dentro del bloque `<graph_enable>` en `/etc/monitorix/monitorix.conf`, reiniciamos el servicio y comprobamos el acceso tanto desde `server4-g4` (`http://192.168.4.103:8080/monitorix`) como desde el navegador del aula (`http://172.26.46.54:8481/monitorix`).
  ![Habilitar gráfica Apache en Monitorix](informe/capturas/fase4/monitorix/image4.png)
  ![Reinicio Monitorix](informe/capturas/fase4/monitorix/image5.png)
  ![Curl desde server4-g4](informe/capturas/fase4/monitorix/image6.png)
  ![Interfaz web Monitorix desde aula](informe/capturas/fase4/monitorix/image7.png)
