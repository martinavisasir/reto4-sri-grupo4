# Memoria Técnica — Reto 4 (Grupo 4)

[⬅ Volver al índice principal](../README.md)

---

## Fase 0: Preparación Inicial y Acceso Remoto (SSH)

### 0.1. Configuración de Red, Hostnames y Zona Horaria

* Configuramos el direccionamiento IP estático en los servidores de la red interna (`192.168.4.0/24`) con puerta de enlace `192.168.4.1`.
  ![Configuración de red 1](capturas/fase0/confred/image1.png)
  ![Configuración de red 2](capturas/fase0/confred/image2.png)
  ![Configuración de red 3](capturas/fase0/confred/image3.png)
  ![Configuración de red 4](capturas/fase0/confred/image4.png)
  ![Configuración de red 5](capturas/fase0/confred/image5.png)
* Asignamos los nombres de equipo (`server1-g4`, `server2-g4`, `server3-g4`, `server4-g4`) para identificar cada nodo de la infraestructura.
  ![Configuración de hostnames](capturas/fase0/hostnames/hostnames.png)
* Ajustamos la zona horaria (`Europe/Madrid`) en todos los servidores para sincronizar los registros y marcas de tiempo de los servicios.
  ![Zona horaria 1](capturas/fase0/zona%20horaria/image1.png)
  ![Zona horaria 2](capturas/fase0/zona%20horaria/image2.png)
  ![Zona horaria 3](capturas/fase0/zona%20horaria/image3.png)
  ![Zona horaria 4](capturas/fase0/zona%20horaria/image4.png)

### 0.2. Servicio SSH y Autenticación por Clave Pública

* Instalamos `openssh-server`, editamos `/etc/ssh/sshd_config` para habilitar `PubkeyAuthentication yes` y `PasswordAuthentication yes`, y reiniciamos el servicio (`sudo systemctl restart ssh`).
  ![Instalación SSH](capturas/fase0/ssh/image1.png)
  ![Editar sshd_config](capturas/fase0/ssh/image2.png)
  ![PubkeyAuthentication yes](capturas/fase0/ssh/image3.png)
  ![PasswordAuthentication yes](capturas/fase0/ssh/image4.png)
  ![Reinicio servicio SSH](capturas/fase0/ssh/image5.png)
* Habilitamos el servicio SSH en pfSense desde su consola mediante la opción `14`.
  ![SSHD en pfSense](capturas/fase0/ssh/image6.png)
* Generamos el par de claves `ed25519` en nuestro PC Windows (`ssh-keygen -t ed25519 -f D:\ssh\id_ed25519`) sin frase de paso.
  ![Generación clave pública Windows](capturas/fase0/ssh/image7.png)
* Accedemos a la interfaz web de pfSense (`172.26.46.54`) y creamos en **Firewall > NAT** las redirecciones desde la red del aula (`172.26.44.0/24`) hacia cada servidor (`2241`–`2244`).
  ![Login pfSense](capturas/fase0/ssh/image8.png)
  ![Dashboard pfSense](capturas/fase0/ssh/image9.png)
  ![Regla NAT SSH](capturas/fase0/ssh/image10.png)
* Habilitamos `PasswordAuthentication yes` en `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` para copiar nuestra clave pública desde PowerShell y verificamos el acceso mediante clave privada (`ssh -i D:\ssh\id_ed25519 -p 2244 alumno@172.26.46.54`).
  ![Permitir password en cloudimg](capturas/fase0/ssh/image11.png)
  ![Copia de clave pública](capturas/fase0/ssh/image12.png)
  ![Acceso por clave pública](capturas/fase0/ssh/image13.png)

---

## Fase 1: Cortafuegos Perimetral pfSense y Reglas NAT

### 1.1. Desactivar DHCP en pfSense

* Antes de desplegar nuestros propios servidores DHCP, deshabilitamos el servicio DHCP integrado en pfSense desde **Services > DHCP Server > LAN**, desmarcando *Enable DHCP server on LAN interface* y guardando los cambios.
  ![Desactivar DHCP en pfSense](capturas/fase1/cambiar%20dhcp/image1.png)

### 1.2. Creación de Reglas de Port Forwarding (NAT)

* Dado que pfSense aplica por defecto una política restrictiva que bloquea el tráfico entrante desde el exterior, creamos en **Firewall > NAT > Port Forward** las reglas necesarias restringiendo el origen a la red del aula (`172.26.44.0/24`):
  * **SSH (TCP):** Puertos `2241` -> `192.168.4.101:22`, `2242` -> `192.168.4.102:22`, `2243` -> `192.168.4.103:22` y `2244` -> `192.168.4.104:22`.
    ![Regla NAT SSH server1](capturas/fase1/creacion%20reglas/image1.png)
    ![Regla NAT SSH server2](capturas/fase1/creacion%20reglas/image2.png)
    ![Regla NAT SSH server3](capturas/fase1/creacion%20reglas/image3.png)
    ![Regla NAT SSH server4](capturas/fase1/creacion%20reglas/image4.png)
  * **Apache2 (TCP):** Puerto `8480` -> `192.168.4.103:80`.
    ![Regla NAT Apache](capturas/fase1/creacion%20reglas/image5.png)
  * **Monitorix (TCP):** Puerto `8481` -> `192.168.4.103:8080`.
    ![Regla NAT Monitorix](capturas/fase1/creacion%20reglas/image6.png)
  * **DNS (TCP/UDP):** Puerto `53` -> `192.168.4.101:53`.
    ![Regla NAT DNS](capturas/fase1/creacion%20reglas/image7.png)
* Aplicamos la configuración con **Apply Changes** y comprobamos la tabla completa de reglas NAT activas.
  ![Tabla completa reglas NAT](capturas/fase1/creacion%20reglas/image8.png)
  ![Confirmación cambios aplicados](capturas/fase1/creacion%20reglas/image9.png)

---

## Fase 2: Servicio DHCP en Alta Disponibilidad (Failover)

* Instalamos `isc-dhcp-server` en `server1-g4`, identificamos la interfaz `eth0` con `ip a` y la configuramos en `/etc/default/isc-dhcp-server`.
  ![Instalación DHCP server1](capturas/fase2/dhcp/image1.png)
  ![Interfaz eth0](capturas/fase2/dhcp/image2.png)
  ![INTERFACESv4 eth0](capturas/fase2/dhcp/image3.png)
* Configuramos `/etc/dhcp/dhcpd.conf` en `server1-g4` como `primary` (`192.168.4.101`, puerto `647`), repartiendo el rango `192.168.4.150`–`192.168.4.199` y entregando el dominio `grupo4reto4.eus` junto con nuestros DNS (`.101` y `.102`). Validamos la sintaxis (`dhcpd -t`), arrancamos el servicio y comprobamos el estado `Both servers normal` en los logs.
  ![Configuración dhcpd.conf primario](capturas/fase2/dhcp/image4.png)
  ![Validación sintaxis primario](capturas/fase2/dhcp/image5.png)
  ![Estado servicio DHCP primario](capturas/fase2/dhcp/image6.png)
  ![Logs failover Both servers normal](capturas/fase2/dhcp/image7.png)
* Repetimos el despliegue en `server2-g4` como `secondary` (`192.168.4.102`, puerto `847`), validando la sintaxis y el arranque del servicio.
  ![Instalación DHCP server2](capturas/fase2/dhcp/image8.png)
  ![INTERFACESv4 server2](capturas/fase2/dhcp/image9.png)
  ![Configuración dhcpd.conf secundario](capturas/fase2/dhcp/image10.png)
  ![Validación sintaxis secundario](capturas/fase2/dhcp/image11.png)
  ![Estado servicio DHCP secundario](capturas/fase2/dhcp/image12.png)
* Añadimos la reserva por dirección MAC para `server4-g4` (`192.168.4.104`) en ambos servidores, configuramos `dhcp4: true` en su Netplan y comprobamos que obtiene la IP reservada y la mantiene incluso tras detener el servicio DHCP en `server1-g4`.
  ![Reserva MAC server4-g4](capturas/fase2/dhcp/image13.png)
  ![Netplan DHCP en server4-g4](capturas/fase2/dhcp/image14.png)
  ![Obtención de IP en server4-g4](capturas/fase2/dhcp/image15.png)
  ![Parada DHCP en server1-g4](capturas/fase2/dhcp/image16.png)
  ![Persistencia de IP tras caída](capturas/fase2/dhcp/image17.png)

---

## Fase 3: Servicio DNS Maestro-Esclavo (`grupo4reto4.eus`)

* Instalamos `bind9`, `bind9utils` y `dnsutils` en ambos servidores DNS y configuramos `/etc/bind/named.conf.options` con los reenviadores `1.1.1.1` y `8.8.8.8`, validación DNSSEC y escucha exclusiva en IPv4.
  ![Instalación BIND9](capturas/fase3/dns/image1.png)
  ![Configuración named.conf.options](capturas/fase3/dns/image2.png)
* Definimos en `/etc/bind/named.conf.local` del maestro las zonas directa (`grupo4reto4.eus`) e inversa (`4.168.192.in-addr.arpa`) autorizando la transferencia y notificación hacia `192.168.4.102`. Creamos `db.grupo4reto4.eus` (registros `A` y alias `CNAME`: `dns1`, `dns2`, `monitor`, `web-proxmox`, `cliente`, `www`) y `db.192.168.4` (registros `PTR`), validándolos con `named-checkconf` y `named-checkzone`.
  ![named.conf.local maestro](capturas/fase3/dns/image3.png)
  ![Zona directa db.grupo4reto4.eus](capturas/fase3/dns/image4.png)
  ![Zona inversa db.192.168.4](capturas/fase3/dns/image5.png)
  ![Comprobación sintaxis DNS](capturas/fase3/dns/image6.png)
* Configuramos `server2-g4` como esclavo (`type slave`) y verificamos la transferencia automática de las zonas en `/var/cache/bind/`. Asignamos ambos DNS (`192.168.4.101` y `192.168.4.102`) y el dominio `grupo4reto4.eus` en Cloud-Init desde Proxmox.
  ![named.conf.local esclavo](capturas/fase3/dns/image7.png)
  ![Archivos transferidos en esclavo](capturas/fase3/dns/image8.png)
  ![Configuración DNS en Cloud-Init](capturas/fase3/dns/image9.png)
* **Pruebas de funcionamiento, sincronización y alta disponibilidad:**
  * Incrementamos el Serial en el maestro a `2026111603`, recargamos el servicio (`sudo systemctl reload bind9`) y verificamos en `server2-g4` tanto el registro `SOA` actualizado como los logs de transferencia (`Transfer status: success`).
    ![Cambio de serial a 2026111603](capturas/fase3/dns/image10.png)
    ![Reload bind9](capturas/fase3/dns/image11.png)
    ![Verificación SOA en esclavo](capturas/fase3/dns/image12.png)
    ![Logs de transferencia en esclavo](capturas/fase3/dns/image13.png)
  * Comprobamos la resolución externa desde un PC Windows del aula hacia la IP WAN de pfSense (`nslookup www.grupo4reto4.eus 172.26.46.54`).
    ![Consulta DNS desde WAN aula](capturas/fase3/dns/image14.png)
  * Verificamos resoluciones inversas (`dig -x`), directas (`pc-win3`, `monitor`) y acceso web por nombre (`curl server3-g4`) en ambos servidores, comprobando finalmente la continuidad del servicio al detener `bind9` en `server1-g4`.
    ![Resolución inversa maestro](capturas/fase3/dns/image15.png)
    ![Resolución inversa esclavo](capturas/fase3/dns/image16.png)
    ![Resolución directa y CNAME maestro](capturas/fase3/dns/image17.png)
    ![Resolución directa y CNAME esclavo](capturas/fase3/dns/image18.png)
    ![Curl por nombre](capturas/fase3/dns/image19.png)
    ![Parada del DNS maestro](capturas/fase3/dns/image20.png)
    ![Resolución con maestro caído](capturas/fase3/dns/image21.png)
    ![Comprobación adicional DNS 1](capturas/fase3/dns/image22.png)
    ![Comprobación adicional DNS 2](capturas/fase3/dns/image23.png)

---

## Fase 4: Servidor Web Apache2 y Monitorización con Monitorix

### 4.1. Servidor Web Apache2 (`server3-g4`)

* Instalamos `apache2`, verificamos que el servicio esté activo y creamos `/var/www/grupo4reto4/index.html`.
  ![Instalación Apache2](capturas/fase4/apache/image1.png)
  ![Estado servicio Apache2](capturas/fase4/apache/image2.png)
  ![Creación web grupo4reto4](capturas/fase4/apache/image3.png)
* Creamos el VirtualHost `/etc/apache2/sites-available/grupo4reto4.conf` con logs personalizados (`grupo4reto4_access.log` y `grupo4reto4_error.log`), lo activamos con `a2ensite`, deshabilitamos `000-default.conf` y validamos la configuración con `apache2ctl configtest` (`Syntax OK`) y `apache2ctl -S`.
  ![VirtualHost grupo4reto4.conf](capturas/fase4/apache/image4.png)
  ![Habilitar sitio](capturas/fase4/apache/image5.png)
  ![Configtest Syntax OK](capturas/fase4/apache/image6.png)
  ![Deshabilitar sitio por defecto](capturas/fase4/apache/image7.png)
  ![Apache2ctl -S](capturas/fase4/apache/image8.png)
* Comprobamos el acceso interno por IP (`curl http://192.168.4.103`) y por nombre (`curl http://web-proxmox.grupo4reto4.eus`), el acceso externo desde el navegador del aula (`http://172.26.46.54:8480`) y los registros de visita en `/var/log/apache2/grupo4reto4_access.log`.
  ![Curl interno por IP](capturas/fase4/apache/image9.png)
  ![Acceso desde navegador aula puerto 8480](capturas/fase4/apache/image10.png)
  ![Curl interno por nombre](capturas/fase4/apache/image11.png)
  ![Revisión configuración VirtualHost](capturas/fase4/apache/image12.png)
  ![Ficheros de log de Apache](capturas/fase4/apache/image13.png)
  ![Registros de acceso](capturas/fase4/apache/image14.png)

### 4.2. Monitorización con Monitorix (`server3-g4`)

* Instalamos `monitorix` y `apache2-utils`, habilitamos el servicio (`sudo systemctl enable --now monitorix`) y comprobamos que responde `HTTP/1.0 200 OK` en el puerto `8080`.
  ![Instalación Monitorix](capturas/fase4/monitorix/image1.png)
  ![Estado servicio Monitorix](capturas/fase4/monitorix/image2.png)
  ![Curl local Monitorix](capturas/fase4/monitorix/image3.png)
* Activamos la gráfica de Apache (`apache = y`) dentro del bloque `<graph_enable>` en `/etc/monitorix/monitorix.conf`, reiniciamos el servicio y comprobamos el acceso tanto desde `server4-g4` (`http://192.168.4.103:8080/monitorix`) como desde el navegador del aula (`http://172.26.46.54:8481/monitorix`).
  ![Habilitar gráfica Apache en Monitorix](capturas/fase4/monitorix/image4.png)
  ![Reinicio Monitorix](capturas/fase4/monitorix/image5.png)
  ![Curl desde server4-g4](capturas/fase4/monitorix/image6.png)
  ![Interfaz web Monitorix desde aula](capturas/fase4/monitorix/image7.png)
