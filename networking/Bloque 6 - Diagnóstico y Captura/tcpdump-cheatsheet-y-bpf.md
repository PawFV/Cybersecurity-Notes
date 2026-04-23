---
title: Tcpdump Cheatsheet y BPF
tags: [networking, tcpdump, bpf, packet-capture, troubleshooting]
aliases: [Tcpdump Basico, BPF Basico, Captura de Paquetes]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[wireshark-display-filters]]"
  - "[[ss-netstat-lsof-procesos-vs-puertos]]"
---

# Tcpdump Cheatsheet y BPF

> [!abstract] TL;DR
> - `tcpdump` captura paquetes en vivo y permite aislar tráfico con **BPF**.
> - Los filtros BPF se aplican **antes** de mostrar o guardar, por eso son eficientes.
> - Saber filtrar por host, red, puerto, flags y dirección ahorra horas de ruido.
> - En troubleshooting serio, `tcpdump` te dice si el paquete salió, llegó y cómo volvió.

## Concepto

`tcpdump` es la navaja suiza de captura en CLI. Mientras Wireshark brilla para análisis profundo visual, `tcpdump` gana cuando necesitás:

- trabajar por SSH;
- capturar en servidores sin entorno gráfico;
- generar `.pcap` para análisis posterior;
- confirmar rápido si un problema es de red, firewall o aplicación.

La pieza clave es **BPF** (Berkeley Packet Filter): un lenguaje de filtros de captura que decide qué paquetes entran a tu vista o archivo.

## Cómo funciona

### Captura sin filtro vs captura útil

Sin filtro, una interfaz productiva te devuelve una manguera abierta:

```bash
sudo tcpdump -i eth0
```

Eso rara vez sirve. Lo valioso es acotar:

```bash
sudo tcpdump -n -i eth0 host 192.168.56.10 and port 443
```

### Modelo mental de BPF

BPF clásico en `tcpdump` filtra por:

- tipo: `host`, `net`, `port`, `portrange`;
- dirección: `src`, `dst`;
- protocolo: `tcp`, `udp`, `icmp`, `arp`;
- operadores lógicos: `and`, `or`, `not`;
- offsets y flags para casos finos.

```ascii
(protocolo) and (quién) and (hacia dónde) and (qué puerto)
```

Ejemplo:

```text
tcp and src host 192.168.56.10 and dst port 443
```

### Capturar vs mostrar

Conviene separar dos ideas:

- **filtro de captura**: decide qué paquetes entran;
- **decodificación / lectura**: cómo los ves luego.

Guardar un `pcap` chico y bien filtrado suele ser mejor que mirar ruido en tiempo real.

> [!tip] `-n` y `-nn`
> Desactivar resolución de nombres evita demoras, tráfico extra y falsas pistas. En análisis inicial, casi siempre conviene `-nn`.

## Comandos / configuración

```bash
# Captura básica sin resolver nombres
sudo tcpdump -nn -i eth0

# Filtrar por host
sudo tcpdump -nn -i eth0 host 192.168.56.10

# Filtrar por red
sudo tcpdump -nn -i eth0 net 192.168.56.0/24

# Filtrar por protocolo y puerto
sudo tcpdump -nn -i eth0 tcp port 443
sudo tcpdump -nn -i eth0 udp port 53

# Solo tráfico saliente a un destino
sudo tcpdump -nn -i eth0 src host 192.168.56.50 and dst host 203.0.113.20

# Guardar pcap
sudo tcpdump -nn -i eth0 -w captura-web.pcap host 203.0.113.20 and port 443

# Leer un pcap
tcpdump -nn -r captura-web.pcap

# Ver payload ASCII
sudo tcpdump -nn -A -i eth0 tcp port 80

# Ver payload hex + ASCII
sudo tcpdump -nn -X -i eth0 host 192.168.56.10

# Capturar SYN
sudo tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Capturar ICMP unreachable
sudo tcpdump -nn -i eth0 'icmp[0] == 3'
```

Chuleta de expresiones útiles:

```text
host 192.168.56.10
src host 192.168.56.10
dst net 10.20.30.0/24
tcp port 22
portrange 1-1024
not arp
tcp and (port 80 or port 443)
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| La app "no conecta" | El paquete nunca sale o vuelve con RST/ICMP. | `tcpdump -nn -i eth0 host 203.0.113.20 and port 443` |
| Hay timeout pero no error explícito | Firewall drop silencioso. | Ver SYN retransmitidos sin respuesta |
| DNS parece lento | Retransmisiones, NXDOMAIN o servidor incorrecto. | `tcpdump -nn -i eth0 udp port 53` |
| No ves tráfico esperado | Interfaz equivocada, VLAN, namespace o filtro mal escrito. | Probar primero `tcpdump -D` y luego un filtro más amplio |
| Captura enorme e inútil | Falta de BPF o demasiado alcance. | Ajustar a host/puerto/protocolo específico |

## Seguridad / ofensiva

### 1. Valor ofensivo y defensivo

`tcpdump` sirve tanto para responder incidentes como para validar un pivot, confirmar beaconing o ver si un payload está exfiltrando datos.

### 2. Riesgos

Capturar paquetes en hosts sensibles puede exponer:

- cookies;
- tokens;
- credenciales en protocolos inseguros;
- metadata de sesiones internas.

> [!danger] Privilegios y alcance
> Una captura de red es material sensible. Guardarla sin control o compartir un `.pcap` crudo puede filtrar mucha más información que el problema original.

### 3. Indicadores prácticos

- SYN repetidos sin SYN-ACK: filtrado o ruta rota.
- DNS repetido con mismo ID o query: pérdida/retransmisión.
- Resets inmediatos: servicio cerrado o middlebox.
- Picos salientes a destinos atípicos: posible beaconing o exfil.

## Relacionado

- [[wireshark-display-filters]]
- [[ss-netstat-lsof-procesos-vs-puertos]]

## Referencias

- `man tcpdump`
- `man pcap-filter`
- tcpdump.org Documentation
- Wireshark Documentation - *Capture Filters*
