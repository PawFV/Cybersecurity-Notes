---
title: NAT: SNAT, DNAT y Masquerade
tags: [networking, l3, l4, nat, pivoting]
aliases: [Port Forwarding, SNAT, DNAT]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[ip-subnetting-cidr]]"
  - "[[iptables-tablas-chains-targets]]"
---

# NAT: SNAT, DNAT y Masquerade

> [!abstract] TL;DR
> - **NAT (Network Address Translation)** parchea el hecho de que nos quedamos sin IPv4. Traduce IPs privadas (adentro) a IPs públicas (afuera).
> - **SNAT (Source NAT):** Modifica la IP de *origen* cuando el tráfico **sale** hacia internet.
> - **Masquerade:** Una forma especial de SNAT para interfaces con IP dinámica (DHCP). "Disfrázate con la IP que tengas en este momento".
> - **DNAT (Destination NAT):** Modifica la IP/puerto de *destino* cuando el tráfico **entra**. Es lo que comúnmente llamás *Port Forwarding*.
> - En Red Team, dominar NAT es el 90% del trabajo de Pivoting a través de redes segmentadas usando iptables o socat.

## Concepto

Imaginá una empresa con 500 empleados, pero la telefónica solo les dio **un único número de teléfono público** (ej. 555-1234).
Adentro, cada empleado tiene un interno (101, 102, 103).

- Si el interno 101 quiere llamar a una pizzería afuera, la centralita (Router/NAT) intercepta la llamada, cambia el origen de "101" a "555-1234", y anota en su libreta: *"Si la pizzería devuelve la llamada a esta hora, pasásela al 101"*. **(SNAT / Masquerade)**.
- Si un cliente de afuera llama al 555-1234 queriendo hablar con Ventas, la centralita intercepta la llamada, y cambia el destino a "Interno 105". **(DNAT / Port Forwarding)**.

Sin la centralita, el cliente de afuera no tiene idea de cómo marcar el "105" directo, porque ese número solo tiene sentido dentro del edificio.

## Cómo funciona

### SNAT (Tráfico Saliente)
Ocurre **después** de que el router decide por dónde sacar el paquete (Post-routing).
```ascii
[PC Interna 192.168.1.50] ─(Llama a 8.8.8.8)─▶ [Router NAT]
                                                   │
 [Router: Cambia Src IP de 192.168.1.50 a IP_PUB] ─┘
                                                   │
                                            [Internet (8.8.8.8)]
```

### DNAT (Tráfico Entrante)
Ocurre **antes** de que el router decida a dónde enviarlo (Pre-routing), para saber a qué PC interna entregarlo.
```ascii
[Internet (Hacker)] ─(Llama a IP_PUB:80)─▶ [Router NAT]
                                                │
 [Router: Cambia Dst IP de IP_PUB a 192.168.1.200] ─┘
                                                │
                                    [Servidor Web 192.168.1.200:80]
```

## Comandos / configuración

En Linux, NAT se configura tradicionalmente con `iptables` en su tabla homónima (`-t nat`), o modernamente con `nftables`. Siempre requiere habilitar el **IP Forwarding** en el kernel primero.

```bash
# 1. Habilitar Forwarding (Ruteo entre interfaces)
echo 1 > /proc/sys/net/ipv4/ip_forward

# ========================================
# Masquerade / SNAT (Salida a Internet)
# ========================================
# Traducir todo el tráfico que sale por eth0 con la IP que tenga eth0 en ese momento.
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# SNAT puro (si eth0 tiene una IP pública estática fija, ej. 203.0.113.5)
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 203.0.113.5

# ========================================
# DNAT / Port Forwarding (Entrada desde Internet)
# ========================================
# Todo lo que entre por eth0 al puerto 80, redirigirlo a 192.168.1.200 al puerto 8080
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.200:8080

# Listar las reglas de NAT activas
iptables -t nat -L -v -n
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Un servidor interno en 192.x.x.x puede navegar por web, pero un ping a 8.8.8.8 falla | Tu regla SNAT/Masquerade especifica `-p tcp` (solo NATea TCP) y no ICMP, o el ISP bloquea ICMP. | `iptables -t nat -S` (revisar que POSTROUTING no esté restringido a TCP). |
| Creaste una regla DNAT para el puerto 80, ves que los paquetes entran, pero no llegan al servidor web | El **IP Forwarding** del kernel está apagado, o el firewall (`FORWARD` chain) está bloqueando el paso de L3. | `sysctl net.ipv4.ip_forward` (debe devolver 1). `iptables -L FORWARD -n`. |
| Las PC internas no pueden acceder al servidor web usando su IP pública (Solo usando la interna) | **Hairpin NAT (NAT Loopback)**. El router recibe un paquete de la LAN destinado a su propia interfaz WAN. Las tablas de NAT se marean si no hay una regla específica para "volver hacia adentro". | En entornos caseros, habilitar NAT Loopback en el router de fibra. |

## Seguridad / ofensiva

Desde la perspectiva del Red Team, NAT es a la vez un escudo accidental y una herramienta ofensiva de primer nivel.

### 1. Pivoting con MASQUERADE (El proxy de los pobres)
Si comprometés una máquina Linux de "borde" (una interfaz mirando a internet, otra a la red interna `10.x.x.x`), podés usarla como pivote L3 para que toda tu infraestructura de ataque llegue adentro.

En lugar de usar SOCKS/Proxychains (L5) que es lento y propenso a errores, activás MASQUERADE en la máquina comprometida. 
1. En tu Kali: `ip route add 10.0.0.0/8 via <IP_Maquina_Comprometida>`
2. En la máquina comprometida: `sysctl net.ipv4.ip_forward=1` y la regla de `MASQUERADE`.

De repente, Nmap desde tu Kali vuela directo a la red 10.x.x.x., sin lidiar con SOCKS proxy. Las víctimas creerán que el escaneo viene de la máquina comprometida (SNAT), no de tu Kali.

### 2. Evasión de NAC con NAT
Si estás físicamente en una red y tu notebook no está autorizada por el control de acceso (NAC / 802.1X), pero hay un teléfono IP autorizado sobre la mesa, podés enchufar un mini-router propio, clonar la MAC del teléfono IP en el puerto WAN del router, y hacer NAT detrás. Toda la red verá el tráfico como si viniera del teléfono IP legítimo.

### 3. NAT como Firewall rudimentario
NAT *no es* seguridad por diseño, pero en la práctica funciona como un Default Deny entrante. A menos que haya una regla DNAT explícita (Port Forwarding), es imposible iniciar una conexión *desde* internet hacia un host con IP privada (RFC 1918) *detrás* del router. 
- *Bypass:* La Reverse Shell (la víctima inicia la conexión saliente) burla esta limitación, ya que el router anota el estado y permite la vuelta del tráfico.

## Relacionado
- [[ip-subnetting-cidr]] (RFC 1918)
- [[iptables-tablas-chains-targets]] (El framework de firewall de Linux subyacente)

## Referencias
- RFC 3022 - *Traditional IP Network Address Translator (Traditional NAT)*
- Man pages: `man iptables-extensions`
