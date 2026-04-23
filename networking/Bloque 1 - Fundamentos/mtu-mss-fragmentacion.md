---
title: MTU, MSS y Fragmentación
tags: [networking, mtu, mss, fragmentacion, vpn]
aliases: [Path MTU, Fragmentación IP]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[osi-vs-tcpip]]"
  - "[[tcp-estados-y-handshake]]"
---

# MTU, MSS y Fragmentación

> [!abstract] TL;DR
> - **MTU (L2):** El paquete IP más grande que puede entrar en una trama Ethernet (Típicamente 1500 bytes).
> - **MSS (L4):** El payload TCP más grande que podés meter (Típicamente 1460 bytes, porque TCP+IP se comen 40 bytes de headers).
> - **Fragmentación:** Si un router recibe un paquete más grande que su MTU de salida, lo corta en pedacitos (Fragmentos IP). El destino final los reensambla.
> - **DF (Don't Fragment):** Flag IP que prohíbe cortar el paquete. Si es muy grande, el router lo tira y avisa (ICMP Fragmentation Needed).
> - En Red Team, jugar con la MTU/Fragmentación es clave para evadir NIDS/IPS, y vital para que los túneles VPN/C2 no se cuelguen.

## Concepto

Imaginá que estás mudando un sofá enorme (el Payload) y tenés que pasarlo por una puerta (el MTU del cable de red). 

- Si el sofá es más grande que la puerta, la única opción es desarmarlo en partes (Fragmentación IP), pasarlas de a una, y que el que recibe arme el sofá de nuevo.
- Si el sofá viene con una etiqueta de "No Desarmar" (Flag DF), y no entra por la puerta, el fletero (Router) deja el sofá en la calle y te llama diciendo: *"Che, conseguite un sofá más chico, este no pasa"* (ICMP Fragmentation Needed).

El **MSS (Maximum Segment Size)** es un acuerdo preventivo. Cuando dos PCs arrancan la llamada TCP (Handshake), se avisan: *"Che, mi puerta mide 1500, así que no me mandes sofás más grandes que 1460"*.

## Cómo funciona

### Las matemáticas de la red estándar
- Trama Ethernet máxima: 1518 bytes.
- Header Ethernet: 14 bytes (Dest MAC, Src MAC, Type) + 4 bytes FCS.
- **MTU (Payload de L2 = Paquete L3):** 1500 bytes.
- Header IPv4: 20 bytes.
- Header TCP: 20 bytes.
- **MSS (Payload de L4 = Datos reales):** 1500 - 20 (IP) - 20 (TCP) = 1460 bytes.

### El problema de los Túneles (VPNs / IPsec / GRE)
Cuando encapsulás tráfico (ej. una VPN), agregás *más* headers. Si tu paquete original era de 1500 bytes, y la VPN le suma 50 bytes de encriptación (IPsec), el nuevo paquete pesa 1550. Ya no entra por el cable físico (MTU 1500). El router se ve forzado a fragmentar, matando la performance de la red, o tira el paquete si tiene DF.

## Comandos / configuración

En Linux, la configuración de MTU se hace por interfaz.

```bash
# ========================================
# Visualización
# ========================================
ip link show eth0    # Verás "mtu 1500" en la línea
# Output:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP...

# ========================================
# Testear Path MTU Discovery (PMTUD)
# ========================================
# Hacemos ping prohibiendo la fragmentación (-M do) y fijando el tamaño del payload (-s)
# 1472 payload + 8 (ICMP header) + 20 (IP header) = 1500
ping -M do -s 1472 8.8.8.8
# Si funciona, el Path MTU es al menos 1500.

# Si probamos con 1473, superamos los 1500 y con DF activado, falla:
ping -M do -s 1473 8.8.8.8
# Output: ping: local error: message too long, mtu=1500

# ========================================
# Ajustar MTU (requiere root)
# ========================================
# Bajar la MTU de una interfaz VPN suele arreglar conexiones "colgadas"
ip link set dev tun0 mtu 1400
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Podés hacer ping, el SSH conecta, pero al tirar un `ls -la` grande o bajar un archivo por HTTP, la conexión se congela (hangs). | **PMTUD Blackhole**. Un router en el medio tiene un MTU menor a 1500. El paquete llega con bandera DF. El router tira el paquete e intenta avisarte por ICMP (Tipo 3, Código 4), pero *un firewall estricto bloqueó todo el ICMP*. Nunca te enterás que tenés que mandar paquetes más chicos. | Bajá la MTU local a mano (`ip link set dev eth0 mtu 1300`) y probá. Si funciona, es un Blackhole. |
| VPN hiper lenta o CPU al 100% en el router | Fragmentación excesiva. Los routers están reensamblando paquetes L3 constantemente. | `tcpdump -n -i eth0 'ip[6] & 0x20 == 0'`. Verás paquetes fragmentados (`frag`). |

## Seguridad / ofensiva

Para el Red Team, la fragmentación es una herramienta de evasión vieja pero a veces efectiva contra NIDS mal configurados (ej. Suricata/Zeek viejos).

### 1. Evasión de NIDS (Fragmentación deliberada)
Si un NIDS busca la firma de ataque "GET /admin/shell.php", podés fragmentar el paquete IP deliberadamente (Nmap `-f`). 
- Fragmento 1: "GET /adm"
- Fragmento 2: "in/shel"
- Fragmento 3: "l.php"

Si el NIDS inspecciona los fragmentos sobre la marcha sin reensamblarlos en memoria (lo cual consume mucha RAM), no verá la firma completa y dejará pasar el ataque. El host destino sí reensambla los paquetes y recibe el ataque íntegro.

### 2. Teardrop Attack (DDoS L3 clásico)
Mandás fragmentos IP malformados donde los "Offsets" (la instrucción de cómo rearmar el sofá) se superponen. (Ej. "El pedazo 2 va del byte 100 al 200, y el pedazo 3 va del 150 al 250"). Los sistemas operativos antiguos entraban en pánico (Kernel Panic) al intentar rearmar esto y crasheaban.

### 3. VPNs / C2 Implosion
Como operador, si levantás un Cobalt Strike Beacon o una Reverse Shell sobre un protocolo tuneado (HTTP/DNS), y el tráfico se "cuelga" a los pocos segundos de establecerse la sesión, 99% de las veces es un problema de MTU. 
La regla de oro en infraestructura ofensiva: **Bajá el MTU de tus interfaces virtuales (tun0/tap0) a 1300 o 1400** para dar espacio a los headers del túnel.

## Relacionado
- [[udp-vs-tcp-cuando-cual]] (Impacto de fragmentación en UDP vs TCP)

## Referencias
- RFC 1191 - *Path MTU Discovery*
- RFC 791 - *Internet Protocol (Fragmentation and Reassembly)*
- Man pages: `man ip-link`
