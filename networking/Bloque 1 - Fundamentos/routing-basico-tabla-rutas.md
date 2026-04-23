---
title: Routing Básico y Tabla de Rutas
tags: [networking, l3, routing, gateway, pivot]
aliases: [Tablas de Ruteo, IP Route, Default Gateway]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[ip-subnetting-cidr]]"
  - "[[nat-snat-dnat-masquerade]]"
---

# Routing Básico y Tabla de Rutas

> [!abstract] TL;DR
> - **Routing (L3)** es el GPS de la red. Determina el salto (Hop) siguiente que debe dar un paquete para llegar a su destino final.
> - Cada host o router tiene una **Tabla de Ruteo**. La regla de oro es: *Match más específico gana*.
> - El **Default Gateway** (`0.0.0.0/0` o `default`) es el "cartero comodín": si la tabla no sabe adónde enviar algo, se lo tira a él.
> - En operaciones Red Team, manipular la tabla de rutas de un pivot (Beachhead) te permite enrutar tráfico de tus herramientas hacia subredes internas profundas sin que el Blue Team note túneles anómalos.

## Concepto

Siguiendo la analogía de la ciudad: tu PC sabe cómo entregar cartas en su propia calle (Subred / L2). Pero si el código postal de destino (IP de Destino) no es de tu calle, no podés ir caminando. Tenés que dejarle la carta a la Oficina de Correos (Router / Default Gateway).

El router tiene un mapa gigante de la ciudad y el país (Tabla de Rutas). Abre el mapa, busca el código postal, y se lo pasa al router de la siguiente provincia (Siguiente Salto / Next Hop). Este proceso se repite hasta que el último router dice: *"Ah, esta es mi calle"*, y hace la entrega final usando ARP (L2).

## Cómo funciona

Una tabla de ruteo típica en Linux (vía `ip route show`) se compone de tres tipos de entradas principales:

1.  **Connected / Link-local:** Redes a las que estás conectado físicamente. *(Ej. Tu propia calle)*.
2.  **Static / Dynamic Routes:** Rutas específicas a subredes particulares que te enseñaron a mano o por protocolos como OSPF/BGP. *(Ej. Para ir al barrio 10.50.0.0/24, tomá la autopista 192.168.1.254)*.
3.  **Default Route:** Si no matchea nada de lo anterior, mandala acá. *(Ej. Para ir a Internet, dásela a 192.168.1.1)*.

### Regla del Longest Prefix Match (El Match más específico)

Si el router tiene dos rutas que coinciden con el destino, *siempre* elegirá la que tenga el CIDR más grande (la red más pequeña y específica).

Ejemplo: Destino `10.50.0.50`.
- Ruta A: `10.0.0.0/8` via Gateway X.
- Ruta B: `10.50.0.0/24` via Gateway Y.
- **Ganador:** Ruta B, porque `/24` es más específico que `/8`.

## Comandos / configuración

La vieja y querida `route -n` está deprecada. Usamos `iproute2`.

```bash
# ========================================
# Visualización
# ========================================
ip route show    # Mostrar tabla de ruteo
# Output típico:
# default via 192.168.1.1 dev eth0 proto dhcp metric 100        <- Default Gateway
# 10.8.0.0/24 dev tun0 proto kernel scope link src 10.8.0.5     <- Ruta VPN (Conectada)
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.50 <- Tu LAN (Conectada)

ip route get 8.8.8.8  # Preguntarle al SO exactamente qué ruta/interfaz usará para esa IP

# ========================================
# Manipulación (Requiere sudo)
# ========================================
# Agregar ruta estática a una red interna profunda
ip route add 10.100.100.0/24 via 192.168.1.254 dev eth0

# Borrar una ruta
ip route del 10.100.100.0/24

# Agregar un nuevo default gateway temporal
ip route add default via 10.8.0.1
```

> [!warning]
> En Linux, los cambios con `ip route` son volátiles (se borran al reiniciar). Para hacerlos persistentes, se usa Netplan, NetworkManager o scripts de `if-up`.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| "Network is unreachable" | No hay Default Gateway ni ruta específica. | `ip route` (buscar la palabra `default`). |
| Ping a internet (`8.8.8.8`) devuelve "Destination Net Unreachable" | El gateway no sabe rutear hacia internet o está caído. | `traceroute 8.8.8.8` (ver en qué salto muere). |
| Latencia intermitente y cortes | Ruteo Asimétrico: El tráfico de ida va por un router A, pero la vuelta vuelve por router B, que corta la conexión por no ver el handshake. | `tcpdump -i eth0 icmp` y revisar las MACs origen/destino en Wireshark. |

## Seguridad / ofensiva

Desde la perspectiva de Red Team, la tabla de ruteo es el mapa del tesoro.

### 1. Pivoting con Rutas Estáticas
Si comprometés un servidor de DMZ (Linux con 2 interfaces: Externa e Interna) y querés escanear la red interna profunda (`10.0.0.0/8`) desde tu notebook (Kali), podés levantar una VPN/Túnel Ligero (como Ligolo-ng o Chisel) entre Kali y la DMZ.

Luego, en Kali, *no* necesitás proxychains para todo. Podés simplemente agregar una ruta:
```bash
# En Kali
ip route add 10.0.0.0/8 dev ligolo-tun
```
A partir de ahí, Nmap y Metasploit funcionan nativamente "como si estuvieras conectado" al switch interno.

### 2. IP Forwarding
Para que una caja Linux comprometida actúe como un router real entre dos interfaces (ej. ens33 y ens34), debes habilitar el forwarding en el kernel:
```bash
sysctl -w net.ipv4.ip_forward=1
```
*(Si no hacés esto, la caja descarta los paquetes que no van dirigidos a sus propias IPs, rompiendo el pivoting en L3).*

### 3. Route Poisoning e ICMP Redirects (MitM)
Un atacante en la red puede enviar paquetes ICMP "Redirect" (Tipo 5) forzando a los hosts a actualizar su tabla de rutas temporalmente. Le dice a la víctima: *"Che, hay un camino más corto hacia internet por acá" (y el "acá" es la IP del atacante)*. Esto permite Hombre-en-el-Medio sin necesidad de inundar la red con ARP Spoofing constante.

## Relacionado
- [[ip-subnetting-cidr]]
- [[nat-snat-dnat-masquerade]]

## Referencias
- RFC 1812 - *Requirements for IP Version 4 Routers*
- Man pages: `man ip-route`
