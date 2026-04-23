---
title: Ethernet y ARP
tags: [networking, l2, ethernet, arp, spoofing]
aliases: [MAC, ARP Poisoning, Layer 2]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[osi-vs-tcpip]]"
  - "[[ip-subnetting-cidr]]"
---

# Ethernet y ARP

> [!abstract] TL;DR
> - **Ethernet (L2)** se encarga de mover bytes *dentro de una misma red física o VLAN* usando direcciones MAC.
> - **ARP (Address Resolution Protocol)** es el puente entre L3 (IP) y L2 (MAC). Pregunta a gritos (Broadcast): *"¿Quién tiene la IP X? Diganme su MAC"*.
> - Las MAC no se enrutan a través de internet; mueren cuando cruzan un router.
> - En pentesting interno, L2 es tu parque de diversiones para interceptar tráfico (MitM) porque ARP por diseño confía en quien le responda.

## Concepto

Imaginá que tu LAN es un salón de clases. La IP es el nombre de la persona (ej. "Juan"), y la MAC es la ubicación de su pupitre. Para mandarle una carta a Juan, no podés poner "Juan" en el sobre y tirarlo al aire; tenés que ir al pupitre exacto. 

**ARP** es cuando te parás en el medio del salón y gritás: *"¿Quién carajo es Juan (IP)? Levantá la mano para ver tu pupitre (MAC)"*. Juan responde, anotás su pupitre en tu libreta (ARP Cache), y de ahí en más le mandás cartas directo.

- **Ethernet** arma el sobre físico (Trama/Frame).
- **ARP** averigua dónde entregarlo localmente.

Si el destino está fuera del salón (otra subred), le mandás la carta al profesor (Default Gateway / Router), quien tiene su propio pupitre (MAC del Router).

## Cómo funciona

### La Trama Ethernet (Frame)
Un paquete IP (L3) se encapsula dentro de una trama Ethernet (L2) antes de salir por el cable o el aire.

```ascii
┌───────┬─────────┬─────────┬─────────┬───────────────┬───────┐
│ Pream │ Dest MAC│ Src MAC │ EtherTyp│ Payload (IP)  │ FCS   │
│ 8 B   │ 6 Bytes │ 6 Bytes │ 2 Bytes │ 46-1500 Bytes │ 4 B   │
└───────┴─────────┴─────────┴─────────┴───────────────┴───────┘
```
- **Dest MAC:** MAC de destino (unicast) o `FF:FF:FF:FF:FF:FF` (broadcast).
- **Src MAC:** Tu propia tarjeta de red.
- **EtherType:** Qué hay adentro (ej. `0x0800` = IPv4, `0x0806` = ARP).
- **FCS:** Frame Check Sequence (para detectar errores en tránsito físico).

### El flujo ARP
1. Host A (10.0.0.5) quiere hacer ping a Host B (10.0.0.10).
2. Host A chequea su **caché ARP**. Si no tiene la MAC de 10.0.0.10, pausa el ping.
3. Host A envía un **ARP Request** (Broadcast a toda la red): *"Who has 10.0.0.10? Tell 10.0.0.5"*.
4. Todos en la red lo reciben. Solo Host B responde con un **ARP Reply** (Unicast a Host A): *"10.0.0.10 is at AA:BB:CC:DD:EE:FF"*.
5. Host A anota esto en su caché y ahora sí, arma la trama Ethernet y manda el ping.

## Comandos / configuración

Para inspeccionar y manipular L2 en Linux, la suite `iproute2` es el estándar. (Olvidate de `ifconfig` y `arp -a`).

```bash
# Ver tu propia MAC y estado del enlace (L2)
ip link show

# Ver la tabla ARP completa (cache local)
ip neigh show
# Output de ejemplo:
# 192.168.1.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE (vivo)
# 192.168.1.5 dev eth0 lladdr aa:bb:cc:dd:ee:ff STALE (en caché pero viejo)
# 192.168.1.9 dev eth0  FAILED (intentamos ARP pero nadie respondió)

# Limpiar la caché ARP para una IP (forzar un nuevo ARP Request)
ip neigh flush dev eth0 192.168.1.10

# Ver tráfico ARP en vivo (Troubleshooting goldmine)
tcpdump -i eth0 arp -n
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| "Destination Host Unreachable" | El host no responde al ARP. La red L3 está bien, pero en L2 nadie contesta. | `ip neigh` (verás la IP en estado FAILED o INCOMPLETE). |
| Latencia alta + pings duplicados | Puede haber dos máquinas con la misma IP, respondiendo ARP intercaladas. | `tcpdump arp` (verás "ARP Reply: MAC_1" y al segundo "ARP Reply: MAC_2" peleando). |
| Caída súbita de conectividad al gateway | ARP Spoofing activo en la red (te robaron la ruta al gateway). | `ip neigh` (comparar la MAC del gateway con la real). |

## Seguridad / ofensiva

ARP fue diseñado en los 80s asumiendo que todos en la red son "de confianza". **No tiene autenticación**. Si alguien te grita un ARP Reply sin que vos hayas preguntado nada (Gratuitous ARP), tu SO lo anota igual.

### 1. ARP Spoofing / Poisoning (MitM)
El ataque por excelencia en LAN. Le decís a la víctima: *"Soy el router (Gateway), mi MAC es la mía"*. Luego le decís al router: *"Soy la víctima, mi MAC es la mía"*.
A partir de ese momento, todo el tráfico entre la víctima y el exterior pasa por tu placa de red.

**Herramientas:** `arpspoof`, `bettercap`, `responder`.

> [!warning] Forwarding
> Si envenenas ARP sin activar IP forwarding en tu Linux (`sysctl -w net.ipv4.ip_forward=1`), vas a actuar como un agujero negro (Blackhole) y le cortarás internet a la víctima, alertando al Blue Team inmediatamente.

### 2. MAC Flooding
Mandás miles de tramas Ethernet con Source MACs falsas y aleatorias muy rápido. La memoria del switch (CAM Table), que mapea MACs a puertos físicos, se llena. 
Cuando un switch se queda sin memoria CAM, entra en modo **Fail-open** (actúa como un Hub tonto) y empieza a retransmitir todo el tráfico por todos los puertos. Ideal para sniffear tráfico que originalmente no iba dirigido a vos.

**Herramienta:** `macof`.

### 3. Evadiendo NAC / Port Security
Si el Blue Team configuró el switch para que solo acepte *una* MAC específica en un enchufe de pared (Port Security o 802.1X básico), podés bypassearlo desconectando el dispositivo legítimo (ej. una impresora IP o teléfono VoIP), copiando su MAC (`ip link set dev eth0 address <MAC_ROBADA>`) y conectando tu notebook al enchufe.

## Relacionado
- [[osi-vs-tcpip]]
- [[ip-subnetting-cidr]]

## Referencias
- RFC 826 - *An Ethernet Address Resolution Protocol*
- Man pages: `man ip-neighbour`
