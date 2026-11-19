# Mapa Lógico de la Red — Grupo 4 (`grupo4reto4.eus`)

[⬅ Volver al índice principal](../README.md)

```mermaid
flowchart TB
    subgraph CENTRO["Red del Centro de Estudios"]
        direction TB

        subgraph AULA["Aula — Equipos Físicos (172.26.44.0/24)"]
            direction LR
            W1["pc-win1<br/>172.26.44.9"]
            W2["pc-win2<br/>172.26.44.10"]
            W3["pc-win3<br/>172.26.44.11"]
            W4["pc-win4<br/>172.26.44.12"]
            W5["pc-win5<br/>172.26.44.13"]
        end

        CORE(("Red Troncal / Enrutamiento<br/>del Centro de Estudios"))

        subgraph VMWARE["Entorno VMware del Centro"]
            VMW["web-vmware<br/>172.16.250.104<br/>Alias: www, web-server, @<br/>Web Principal (HTTPS :443)"]
        end

        subgraph PVE["Servidor Físico Proxmox VE (pve — 172.26.46.240)"]
            direction TB
            FW["pfSense (gateway / pfsense)<br/>WAN (vtnet0): 172.26.46.54/24<br/>LAN (vtnet1): 192.168.4.1/24<br/>──────────────────────────<br/>Reglas NAT (Origen: 172.26.44.0/24):<br/>• TCP 2241-2244 → SSH (.101-.104 :22)<br/>• TCP 8480 → Apache2 (.103 :80)<br/>• TCP 8481 → Monitorix (.103 :8080)<br/>• TCP/UDP 53 → DNS (.101 :53)"]

            subgraph LAN["Red Virtual Interna Grupo 4 (192.168.4.0/24)"]
                S1["server1-g4 (dns1)<br/>192.168.4.101<br/>DNS Maestro + DHCP Primary (:647)"]
                S2["server2-g4 (dns2)<br/>192.168.4.102<br/>DNS Esclavo + DHCP Secondary (:847)"]
                S3["server3-g4 (monitor / web-proxmox)<br/>192.168.4.103<br/>Apache2 (:80) + Monitorix (:8080)"]
                S4["server4-g4 (cliente)<br/>192.168.4.104 (Reserva MAC)<br/>Cliente Ubuntu"]
                POOL["Pool Dinámico DHCP<br/>192.168.4.150 – 192.168.4.199"]
            end
        end

        AULA <--> CORE
        CORE -- "HTTPS (www.grupo4reto4.eus)" --> VMWARE
        CORE -- "SSH (2241-2244) | HTTP (8480, 8481) | DNS (53)" --> FW

        FW --- S1
        FW --- S2
        FW --- S3
        FW --- S4

        S1 <-- "Transferencia Zona (AXFR/NOTIFY) & DHCP Failover" --> S2
        S1 -. "Asignación IP fija (MAC)" .-> S4
        S1 -. "Gestión rango dinámico" .-> POOL
        S2 -. "Respaldo rango dinámico" .-> POOL
    end
```

## Descripción de la Topología Lógica

* **Subred del Aula (`172.26.44.0/24`):** Alberga los 5 equipos físicos Windows (`pc-win1` a `pc-win5`, IPs `.9` a `.13`) desde los que se administra la infraestructura y se validan los servicios[cite: 1].
* **Red Troncal del Centro:** Interconecta la subred del aula con las redes donde se alojan los hipervisores del centro (`172.26.46.0/24` para Proxmox VE y `172.16.250.0/24` para VMware)[cite: 1, 4].
* **Servidor Proxmox VE (`172.26.46.240`) y Red Interna (`192.168.4.0/24`):** El nodo Proxmox aloja tanto la máquina virtual de **pfSense** (con interfaz WAN `vtnet0` en `172.26.46.54/24` e interfaz LAN `vtnet1` en `192.168.4.1/24`) como los cuatro servidores Ubuntu del Grupo 4 (`server1-g4` a `server4-g4`), aislados tras el cortafuegos[cite: 1, 4].
