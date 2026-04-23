---
title: Network Namespaces
tags: [networking, linux, namespaces, containers, isolation]
aliases: [netns, Linux netns, Namespaces de red]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[interfaces-ip-link-addr-route]]"
  - "[[iproute2-suite-ss-ip-bridge]]"
  - "[[bonding-bridging-vlans]]"
---

# Network Namespaces

> [!abstract] TL;DR
> - Un network namespace (`netns`) es una pila de red aislada dentro del mismo kernel.
> - Cada namespace tiene sus propias interfaces, tabla de rutas, ARP/neighbor cache, reglas, firewall y sockets.
> - Los contenedores usan este mecanismo para no compartir directamente el plano de red del host.
> - Para labs, troubleshooting y Red Team, `netns` permite simular redes enteras en una sola máquina.

## Concepto

Un network namespace separa el contexto de red. Lo que en el host parece una única máquina, dentro del kernel pueden ser varias "máquinas lógicas" con su propio stack L2/L3/L4.

Eso implica aislamiento de:

- interfaces
- direcciones IP
- rutas
- vecinos ARP/NDP
- reglas de routing
- sockets
- iptables/nftables según namespace

La excepción importante es que todos siguen compartiendo **el mismo kernel**. No es virtualización completa; es aislamiento de recursos de red.

Metáfora útil, pero acotada: pensalo como varios cuartos dentro del mismo edificio. Cada cuarto tiene su tablero eléctrico de red, pero el edificio sigue siendo uno.

## Cómo funciona

Cuando creás un namespace nuevo, arranca casi vacío. Normalmente solo tiene `lo` apagada. Para conectarlo al resto del sistema, se suele usar un par **veth**: dos extremos virtuales enchufados espalda con espalda.

```ascii
[ host namespace ] veth-host <====> veth-ns [ app namespace ]
       |                                       |
    bridge/br0                              eth0
       |                                       |
   otra red / uplink                      ruta default
```

Flujo típico:

1. Se crea `ns1`.
2. Se crea par `veth-host` / `veth-ns`.
3. Un extremo queda en el host, el otro se mueve a `ns1`.
4. Dentro de `ns1`, `veth-ns` se renombra a `eth0`, se le asigna IP y se levanta.
5. Se agrega ruta default dentro del namespace.

Como cada namespace tiene sockets propios, un `ss -ltnp` dentro de `ns1` no muestra lo mismo que el del host.

> [!note]
> Lo que "desaparece" al mover una interfaz a otro namespace no se destruye: cambia de contexto. Error clásico de troubleshooting en contenedores.

## Comandos / configuración

```bash
# Crear namespace
sudo ip netns add ns1

# Ver namespaces
ip netns list

# Crear par veth
sudo ip link add veth-host type veth peer name veth-ns

# Mover un extremo al namespace
sudo ip link set veth-ns netns ns1

# Configurar lado host
sudo ip addr add 192.0.2.1/24 dev veth-host
sudo ip link set veth-host up

# Configurar lado namespace
sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns1 ip link set veth-ns name eth0
sudo ip netns exec ns1 ip addr add 192.0.2.10/24 dev eth0
sudo ip netns exec ns1 ip link set eth0 up
sudo ip netns exec ns1 ip route add default via 192.0.2.1

# Ejecutar comandos dentro del namespace
sudo ip netns exec ns1 ping -c 3 192.0.2.1
sudo ip netns exec ns1 ss -ltnp
```

Si querés que `ns1` salga hacia otra red del host, normalmente necesitás:

- `ip_forward=1`
- NAT o forwarding en el host
- o un bridge/uplink real

Ejemplo mínimo de forwarding/NAT:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 192.0.2.0/24 -o eth0 -j MASQUERADE
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Dentro del namespace no funciona ni `ping` al gateway | `lo` o la veth siguen `DOWN` | `ip netns exec ns1 ip link` |
| El namespace llega al host pero no a Internet | Falta forwarding/NAT o ruta de retorno | `sysctl net.ipv4.ip_forward`, `iptables -t nat -S` |
| La interfaz "desapareció" del host | Fue movida a otro namespace | `ip netns exec <ns> ip link`, `ip netns list` |
| El servicio escucha, pero no desde afuera | Está bindeado dentro de otro namespace | `ip netns exec <ns> ss -ltnp` |
| Limpiaste un namespace y quedó basura | veth huérfana, reglas o procesos vivos | `ip link`, `ip netns pids <ns>` |

> [!tip]
> Para labs complejos, nombrá siempre los extremos veth de forma explícita. `veth0@if12` sirve poco cuando tenés diez namespaces corriendo.

## Seguridad / ofensiva

Los namespaces son ubicuos en contenedores, runners CI/CD y entornos multi-tenant. Para ofensiva y defensa, eso tiene implicancias concretas.

### Desde Red Team

- Un escape parcial a shell dentro de contenedor puede dejarte en un `netns` con visibilidad limitada.
- Enumerar rutas e interfaces del contenedor te dice si hay acceso a redes laterales que el host también toca.
- Bridges como `docker0`, pares `veth*` y reglas NAT muestran cómo pivotear o dónde inspeccionar tráfico.

### Desde defensa

- Aislar workloads en namespaces distintos reduce exposición accidental entre servicios.
- No confundir namespace con frontera de seguridad absoluta: si el proceso tiene capacidades elevadas (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`), el impacto cambia drásticamente.
- Logs y monitoreo tienen que contemplar que el socket visible en un namespace puede no verse igual desde el host.

```bash
# Vista útil de hunting en host Linux
ip -br link
ip netns list
ss -tulpen
```

> [!warning]
> Dar `CAP_NET_ADMIN` a un contenedor para "resolver rápido" un problema de red suele abrir una superficie operativa innecesaria. Es una concesión fuerte, no un parche inocente.

> [!danger]
> Un namespace no reemplaza segmentación real. Si todo termina bridgeado a la misma red plana del host, el aislamiento es más administrativo que defensivo.

## Relacionado

- [[interfaces-ip-link-addr-route]] (objetos básicos del stack Linux)
- [[iproute2-suite-ss-ip-bridge]] (herramientas para operar namespaces)
- [[bonding-bridging-vlans]] (cómo interconectarlos a L2)

## Referencias

- `man ip-netns`
- `man ip-link`
- [Linux kernel documentation - network namespaces](https://docs.kernel.org/networking/namespaces.html)
- [Linux kernel documentation - veth](https://docs.kernel.org/networking/veth.html)
- [OCI runtime spec - Linux namespaces](https://github.com/opencontainers/runtime-spec/blob/main/config-linux.md)
