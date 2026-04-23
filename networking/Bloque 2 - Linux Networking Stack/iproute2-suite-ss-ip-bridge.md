---
title: Suite iproute2, ss, ip y bridge
tags: [networking, linux, iproute2, ss, bridge]
aliases: [iproute2 suite, ss e ip, Herramientas Linux networking]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[interfaces-ip-link-addr-route]]"
  - "[[bonding-bridging-vlans]]"
  - "[[ss-netstat-lsof-procesos-vs-puertos]]"
---

# Suite iproute2, ss, ip y bridge

> [!abstract] TL;DR
> - `iproute2` es la caja de herramientas moderna para inspeccionar y manipular networking en Linux.
> - `ip` cubre interfaces, direcciones, rutas, vecinos, policy routing y namespaces; `ss` inspecciona sockets; `bridge` administra switching L2 en bridges Linux.
> - Si seguís usando `ifconfig`, `route`, `arp` o `netstat`, estás operando con herramientas obsoletas o incompletas.
> - Para troubleshooting serio, la combinación mínima suele ser: `ip`, `ss`, `bridge`, `tcpdump`.

## Concepto

`iproute2` no es un binario único sino una suite orientada a netlink. Reemplaza el set histórico de herramientas de `net-tools` porque expone capacidades que el kernel Linux moderno realmente usa: VRF, namespaces, VLAN filtering, qdisc, policy routing, VXLAN, bridges y más.

Mapa mental útil:

- **`ip`**: estado y configuración del plano de red del host.
- **`ss`**: quién escucha, quién conecta, y con qué estado.
- **`bridge`**: forwarding L2, FDB, VLANs y puertos de bridge.

```ascii
Plano de sockets      -> ss
Plano L3/L2 del host  -> ip
Plano switching Linux -> bridge
Captura de paquetes   -> tcpdump
```

> [!note]
> Muchas veces el problema no es "de red" en abstracto. Es de plano equivocado. `ss` mira sockets; `ip route` mira forwarding; `bridge fdb` mira aprendizaje MAC. Si mezclás planos, perdés tiempo.

## Cómo funciona

### `ip`: vista del kernel sobre red local

`ip` consulta y modifica objetos del stack: links, addresses, routes, neigh, rule, netns. No opera sobre archivos de config sino sobre el estado vivo del kernel.

### `ss`: sockets y estados

`ss` lee información de sockets desde el kernel y muestra:

- puertos en listen
- conexiones establecidas
- timers TCP
- procesos asociados
- métricas de retransmisión y colas

Eso lo vuelve mucho más útil que `netstat` para incident response y diagnóstico fino.

### `bridge`: switching software

Cuando un host Linux actúa como bridge, el kernel aprende MACs por puerto y decide forwarding igual que un switch básico. `bridge` permite ver:

- FDB (forwarding database)
- puertos miembros
- VLAN filtering
- STP y parámetros relacionados

## Comandos / configuración

```bash
# ===== ip =====
ip -br link
ip -br addr
ip route
ip neigh
ip rule show

# Rutas por tabla
ip route show table main
ip route show table local

# ===== ss =====
ss -tulpen
ss -tan state established
ss -o state time-wait
ss -ltnp

# ===== bridge =====
bridge link
bridge fdb show
bridge vlan show

# Crear un bridge simple y sumar una interfaz
sudo ip link add br0 type bridge
sudo ip link set dev eth1 master br0
sudo ip link set dev br0 up
sudo ip link set dev eth1 up
```

Ejemplo de lectura rápida:

```text
$ ss -ltnp
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      128    0.0.0.0:22        0.0.0.0:*         users:(("sshd",pid=812,fd=3))
LISTEN 0      511    127.0.0.1:8080    0.0.0.0:*         users:(("python3",pid=1440,fd=5))
```

Esto te deja responder enseguida:

- qué servicio escucha
- en qué IP exacta
- si expone localhost o toda la red
- qué proceso real lo abrió

> [!tip]
> `ip -br` y `ss -tulpen` son dos comandos de altísimo rendimiento cognitivo. Dan una foto rápida y suficiente para decidir por dónde seguir.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| "El puerto está abierto" pero nadie conecta | Servicio escuchando solo en `127.0.0.1` o `::1` | `ss -ltnp` |
| La interfaz slave no forwardea tráfico | No está anexada al bridge o quedó `DOWN` | `bridge link`, `ip link show` |
| Hay reachability local pero no salidas específicas | Policy routing o tabla distinta a `main` | `ip rule show`, `ip route show table all` |
| Un vecino cambia de MAC seguido | Flapping, ARP spoofing o bridge mal armado | `ip neigh`, `bridge fdb show` |
| Conexiones lentas o colas crecientes | Retransmisiones, backlog o app saturada | `ss -ti`, `ss -m` |

> [!warning]
> `ss` muestra sockets del host; no demuestra que exista reachability extremo a extremo. Un `LISTEN` no implica que un firewall, una policy route o un bridge estén dejando pasar tráfico.

## Seguridad / ofensiva

Desde Red Team y DevSecOps, esta suite es valiosa porque permite **enumerar sin ruido**. Antes de escanear la red, conviene entender el host.

### Enumeración local útil

```bash
ip -br addr
ip route
ss -tulpen
bridge fdb show
```

Con eso podés detectar:

- sockets de administración expuestos solo en loopback
- contenedores y bridges locales (`docker0`, `cni0`, `br-*`)
- segmentos internos alcanzables por rutas no evidentes
- interfaces de túneles (`tun0`, `wg0`, `gre0`)

### Bridge Linux como pivote

Si comprometés un host con dos NICs o acceso a namespaces/contenedores, un bridge mal controlado puede convertirse en un punto de intercepción, forwarding o expansión lateral dentro del mismo dominio L2.

### Indicadores defensivos

- FDB anómala o MACs moviéndose entre puertos virtuales.
- Servicios escuchando en `0.0.0.0` sin necesidad operativa.
- Policy routing inesperado.
- Gran cantidad de sockets `TIME-WAIT`, `SYN-SENT` o colas altas, indicador de abuso o problemas de app.

> [!danger]
> Crear bridges o mover interfaces a un `master` en un host remoto puede dejarte sin conectividad de administración si no entendés cómo está resuelto el plano de IPs y el path de retorno.

## Relacionado

- [[interfaces-ip-link-addr-route]] (fundamentos de `ip`)
- [[bonding-bridging-vlans]] (diseños L2 sobre Linux)
- [[ss-netstat-lsof-procesos-vs-puertos]] (inspección de puertos y procesos)

## Referencias

- `man ip`
- `man ss`
- `man bridge`
- [iproute2 documentation](https://wiki.linuxfoundation.org/networking/iproute2)
- [Linux kernel documentation - bridge](https://docs.kernel.org/networking/bridge.html)
- [Linux kernel documentation - netlink](https://docs.kernel.org/userspace-api/netlink/intro.html)
