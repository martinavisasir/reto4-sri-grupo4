# Gestión del proyecto · Reto 4 · Grupo 4

## Reparto de tareas

| Fase | Contenido | Responsable |
|---|---|---|
| 0 | Infraestructura base: IP estática con Cloud-Init, hostnames, SSH (contraseña y clave pública), NTP | Asier |
| 1 | Firewall pfSense: DHCP deshabilitado en LAN, port forwards (web, Monitorix, DNS) y denegar por defecto | Martin |
| 1 | Port forwards SSH (2241–2244) | Martin |
| 2 | DHCP con failover (server1 primario, server2 secundario) y reserva MAC de server4 | Asier |
| 3 | DNS maestro-esclavo con BIND9 (zonas directa e inversa, A y CNAME) | Martin |
| 4 | Apache2 (VirtualHost `www.grupo4reto4.eus`) y Monitorix en server3 | Asier |
| GitHub | Repositorio, README, informe, `/config`, `/scripts`, `/diagramas` | Asier / Martin |
