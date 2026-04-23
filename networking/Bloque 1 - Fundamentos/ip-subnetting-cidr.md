---
title: IP, Subnetting y CIDR
tags: [networking, l3, ipv4, subnetting, cidr]
aliases: [Subredes, Máscaras, CIDR]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[osi-vs-tcpip]]"
  - "[[routing-basico-tabla-rutas]]"
---

# IP, Subnetting y CIDR

> [!abstract] TL;DR
> - **IPv4** es una dirección de 32 bits (4 octetos). Define *quién sos* en la red (L3).
> - **Subnet Mask** define *dónde estás* (cuál es el límite de tu red local).
> - **CIDR** (Classless Inter-Domain Routing) es la notación moderna (`/24`, `/26`) que reemplazó a las rígidas clases A, B y C.
> - Si dos IPs están en la misma subnet, hablan directo (ARP/MAC L2). Si están en subnets distintas, necesitan un Router (L3).

## Concepto

Imaginá que tu IP (`192.168.1.50`) es la dirección de tu casa: "Calle Falsa 123". El problema es que "Calle Falsa" es larguísima y el cartero (Router) necesita saber si el número 123 está en *tu* cuadra o en la cuadra de al lado. 

La **Máscara de Subred** (Subnet Mask) es el límite de la cuadra. Le dice a tu computadora: *"Todo lo que empiece con 192.168.1 pertenece a esta cuadra (Network ID). Si querés mandarle algo al 192.168.2.10, dáselo al cartero (Default Gateway) porque está en otra cuadra"*.

- **Network ID:** El nombre de la cuadra (ej. `192.168.1.0`). Nadie puede tener esta IP.
- **Broadcast Address:** El megáfono de la cuadra (ej. `192.168.1.255`). Si mandás un paquete acá, le llega a todas las casas de la cuadra a la vez.
- **Host IPs:** Las casas usables (`192.168.1.1` al `.254`).

## Cómo funciona (Subnetting)

Una IPv4 son 32 bits (unos y ceros) divididos en 4 bloques de 8 bits (octetos). La notación CIDR `/XX` te dice *cuántos de esos 32 bits están fijos (son la red)*. Los bits restantes son para los hosts (las casas).

Ejemplo clásico: `192.168.1.50/24`

```ascii
IP:       11000000 . 10101000 . 00000001 . 00110010 (192.168.1.50)
Máscara:  11111111 . 11111111 . 11111111 . 00000000 (255.255.255.0)
          |--------- 24 bits de Red -------|-- 8 --|
                                             Hosts
```
- Bits de Host = `32 - 24 = 8 bits`.
- Casas posibles = $2^8 - 2$ (Red y Broadcast) = $256 - 2 = 254$ IPs usables.

Si cortás la cuadra por la mitad (`/25`):
- Bits de red: 25.
- Bits de host: `32 - 25 = 7`.
- Casas posibles: $2^7 - 2 = 126$ IPs usables.
- Subnet 1: `192.168.1.0/25` (Hosts 1 al 126).
- Subnet 2: `192.168.1.128/25` (Hosts 129 al 254).

> [!tip] La regla del 2
> Cada vez que sumás +1 al CIDR (ej. de `/24` a `/25`), dividís la red en la mitad exacta. De 254 IPs pasás a dos redes de 126 IPs.

## Comandos / configuración

En Linux, la suite `iproute2` y utilidades de cálculo rápido (`ipcalc` o `sipcalc`) son tus amigas.

```bash
# Ver tu IP y máscara actual en formato CIDR
ip addr show eth0
# Output: inet 192.168.1.50/24 brd 192.168.1.255 scope global eth0

# Calcular rápidamente los límites de una red (Requiere instalar ipcalc)
ipcalc 192.168.1.50/26
# Output:
# Address:   192.168.1.50
# Netmask:   255.255.255.192 = 26
# Network:   192.168.1.0/26      <- (Donde empieza)
# HostMin:   192.168.1.1
# HostMax:   192.168.1.62        <- (Donde termina)
# Broadcast: 192.168.1.63
# Hosts/Net: 62

# Asignar una IP temporalmente a una interfaz (muy útil en Pivoting)
sudo ip addr add 10.0.0.100/24 dev eth1
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Podés hacer ping a 8.8.8.8 pero no a tu compañero de al lado | Máscara incorrecta. Te configuraste un `/30` en vez de un `/24`, tu PC cree que tu compañero está "en otra cuadra" y se lo manda al router. | `ip addr show` (comparar CIDRs). |
| Broadcast storms (la red está inusable) | Subnet excesivamente grande (ej. `/8` o `/16`) en un solo dominio de broadcast (VLAN plana). El tráfico ARP y DHCP ahoga los switches. | `tcpdump -n -i eth0 broadcast` (verás saturación constante). |
| IP Conflict (IP ya en uso) | Dos hosts configurados estáticamente con la misma IP en la misma subnet. | ARP spoofing no intencional. `arping 192.168.1.50`. |

## Seguridad / ofensiva

Desde la perspectiva de Red Team, entender cómo el Blue Team segmentó sus subredes dicta tu estrategia de Movimiento Lateral y Descubrimiento.

### 1. Ping Sweeps (`nmap -sn`)
Cuando comprometés un host inicial (Beachhead), lo primero es ver "quién más vive en la cuadra". Como todos en la misma subnet se hablan directamente (L2), un ping sweep es rapidísimo y no pasa por el firewall corporativo (solo por switches L2).

```bash
nmap -sn 192.168.1.0/24
```
> [!warning] Detección EDR
> Tirar un `/24` entero muy rápido puede ser detectado por EDRs modernos. Mejor segmentar el escaneo o usar pausas (T2/T1), o escanear solo los `/26` o `/28` más probables (ej. los primeros 20 hosts donde suelen estar servidores y gateways).

### 2. IP Spoofing (Falsificación de Origen)
Si conocés el CIDR de una "red de confianza" (ej. VLAN de Administradores `10.10.10.0/24`), podés forjar paquetes UDP/ICMP diciendo que venís de ahí. El router perimetral te los dejará pasar si no tiene uRPF (Unicast Reverse Path Forwarding) activado.

### 3. Evadiendo segmentación (VLAN Hopping)
Una técnica vieja pero a veces funcional. Si el puerto del switch donde estás conectado está mal configurado como "Trunk" (acepta tráfico de múltiples VLANs), podés inyectar paquetes con el Tag 802.1Q de otra subnet.

## Relacionado
- [[ethernet-y-arp]] (Cómo hablan los hosts en la misma subnet)
- [[routing-basico-tabla-rutas]] (Cómo hablan hosts de diferentes subnets)

## Referencias
- RFC 4632 - *Classless Inter-domain Routing (CIDR)*
- RFC 1918 - *Address Allocation for Private Internets*
- Man pages: `man ip-address`
