# Mapa Lógico de la Red — Grupo 4 (`grupo4reto4.eus`)

[⬅ Volver al índice principal](../README.md)

```mermaid
flowchart TB
    subgraph AULA["Red Física del Aula (172.26.44.0/24)"]
        direction LR
        W1["pc-win1<br/>172.26.44.9"]
        W2["pc-win2<br/>172.26.44.10"]
        W3["pc-win3<br/>172.26.44.11"]
        W4["pc-win4<br/>172.26.44.12"]
        W5["pc-win5<br/>172.26.44.13"]
    end

    subgraph VMWARE["Entorno Externo VMware"]
        VMW["web-vmware<br/>IP: 172.16.250.104<br/>Alias: www, web-server, @<br/>Servicio: Web Principal (HTTPS :443)"]
    end

    subgraph PFS["Firewall Perimetral pfSense (gateway / pfsense)"]
        FW["Interfaz WAN: 172.26.46.54/24<br/>Interfaz LAN: 192.168.4.1/24<br/>──────────────────────────<br/>Reglas NAT (Origen: 172.26.44.0/24):<br/>• TCP 2241-2244 → SSH (.101-.104 :22)<br/>• TCP 8480 → Apache2 (.103 :80)<br/>• TCP 8481 → Monitorix (.103 :8080)<br/>• TCP/UDP 53 → DNS (.101 :53)"]
    end

    subgraph PROXMOX["Red Interna Proxmox VE — Grupo 4 (192.168.4.0/24)"]
        S1["server1-g4 (dns1)<br/>IP: 192.168.4.101<br/>• DNS Maestro (BIND9)<br/>• DHCP Primary (Failover :647)"]
        S2["server2-g4 (dns2)<br/>IP: 192.168.4.102<br/>• DNS Esclavo (BIND9)<br/>• DHCP Secondary (Failover :847)"]
        S3["server3-g4 (monitor / web-proxmox)<br/>IP: 192.168.4.103<br/>• Apache2 (:80)<br/>• Monitorix (:8080)"]
        S4["server4-g4 (cliente)<br/>IP: 192.168.4.104 (Reserva MAC)<br/>• Cliente Ubuntu"]
        POOL["Pool Dinámico DHCP<br/>Rango: 192.168.4.150 – 192.168.4.199"]
    end

    AULA -- "Acceso Web (www.grupo4reto4.eus)" --> VMWARE
    AULA -- "SSH (2241-2244) | HTTP (8480, 8481) | DNS (53)" --> FW

    FW --- S1
    FW --- S2
    FW --- S3
    FW --- S4

    S1 <-- "Transferencia Zona (AXFR/NOTIFY) & DHCP Failover" --> S2
    S1 -. "Asignación IP fija (MAC)" .-> S4
    S1 -. "Gestión rango dinámico" .-> POOL
    S2 -. "Respaldo rango dinámico" .-> POOL
```

## Resumen de Flujos Lógicos

* **Desde el Aula (`172.26.44.0/24`):** Los 5 equipos Windows (`pc-win1` a `pc-win5`) tienen acceso filtrado por el cortafuegos `pfSense` (`172.26.46.54`) hacia los puertos redirigidos por NAT (`2241–2244` para administración SSH, `8480` para Apache2, `8481` para Monitorix y `53` para resolución DNS), además de acceso directo al servidor web del grupo en VMware (`172.16.250.104`).
* **En la Red Interna Proxmox (`192.168.4.0/24`):** `pfSense` (`192.168.4.1`) actúa como puerta de enlace con su servicio DHCP deshabilitado. La asignación de direcciones IP recae sobre el clúster `isc-dhcp-server` en alta disponibilidad entre `server1-g4` (`.101`) y `server2-g4` (`.102`), que también operan como servidores DNS maestro y esclavo autoritativos para el dominio `grupo4reto4.eus`.
