---
title: Traceroute, MTR y PathPing
tags: [networking, traceroute, mtr, pathping, troubleshooting]
aliases: [Diagnostico de Ruta, Saltos de Red, MTR Basico]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[routing-basico-tabla-rutas]]"
  - "[[tcpdump-cheatsheet-y-bpf]]"
---

# Traceroute, MTR y PathPing

> [!abstract] TL;DR
> - `traceroute`, `mtr` y `pathping` ayudan a entender por qué camino llega el tráfico y dónde aparece latencia o pérdida.
> - No muestran "la ruta exacta de la app" con precisión absoluta, sino cómo responden routers intermedios a probes específicos.
> - `traceroute` da una foto; `mtr` combina ruta y estadísticas; `pathping` hace algo parecido en Windows.
> - Un salto que no responde ICMP no significa necesariamente que el tráfico real esté roto.

## Concepto

Estas herramientas trabajan explotando el campo **TTL** de IP. En vez de mandar un paquete común y esperar destino final, mandan probes con TTL creciente:

- TTL 1: el primer router lo decrementa a 0 y responde.
- TTL 2: responde el segundo.
- TTL 3: responde el tercero.

Y así sucesivamente hasta llegar al destino o agotar saltos.

Sirven para responder preguntas como:

- ¿por dónde sale mi tráfico?
- ¿en qué tramo aparece latencia?
- ¿hay pérdida consistente en un hop o más adelante?
- ¿estoy pegándole al destino esperado o a otro camino?

## Cómo funciona

### Traceroute clásico

`traceroute` envía probes con TTL creciente. Según plataforma y flags, puede usar UDP, ICMP o TCP.

```ascii
Origen ---- R1 ---- R2 ---- R3 ---- Destino
 TTL=1       x
 TTL=2              x
 TTL=3                     x
 TTL=4                            respuesta final
```

Cada router intermedio suele devolver `ICMP Time Exceeded`.

### MTR

`mtr` es básicamente `traceroute + ping` en loop. No se queda con una sola medición por salto: toma varias muestras y calcula pérdida y latencia promedio.

Eso lo vuelve mucho más útil para problemas intermitentes.

### PathPing

En Windows, `pathping` combina descubrimiento de ruta con medición sostenida de pérdida por hop.

> [!tip] Elegí la sonda correcta
> Si ICMP está filtrado, un `traceroute` por ICMP puede engañarte. En varios escenarios conviene probar también con TCP al puerto real del servicio.

## Comandos / configuración

Linux:

```bash
# Traceroute clásico
traceroute 203.0.113.20

# Usar ICMP
traceroute -I 203.0.113.20

# Usar TCP al puerto 443
traceroute -T -p 443 203.0.113.20

# MTR resumido
mtr -rw 203.0.113.20

# MTR interactivo
mtr 203.0.113.20
```

Windows:

```powershell
tracert 203.0.113.20
pathping 203.0.113.20
```

Ejemplo de lectura rápida en `mtr`:

```text
Loss%   Snt   Last   Avg  Best  Wrst StDev
 0.0%    50    1.2   1.4   1.1   2.0   0.2  gw.local
 0.0%    50    4.1   4.5   4.0   6.8   0.5  isp-edge
12.0%    50   20.0  24.1  18.2  60.3   8.1  upstream-x
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Un hop muestra `* * *` pero el destino responde | Ese router no responde probes de control, pero sí enruta tráfico real. | Confirmar con `mtr` y test al servicio real |
| La pérdida aparece en un hop intermedio pero no continúa | Rate limiting de ICMP en ese router, no pérdida real de tránsito. | Mirar hops siguientes y destino |
| La latencia sube de golpe a partir de un punto y se mantiene | Congestión, enlace distante o cambio de path. | `mtr -rw destino` |
| `traceroute` no llega pero la web sí | El método de probe está filtrado, no necesariamente el servicio. | Probar `-T -p 443` |
| Un servicio falla desde una red pero no otra | Diferente enrutamiento, ACL o salida upstream. | Comparar `mtr` desde ambos orígenes |

> [!warning] ICMP rate limiting
> Muchos routers priorizan forwarding por encima de responder ICMP. Un hop "lento" en traceroute puede ser solo lento para contestarte a vos, no para enrutar tráfico de usuarios.

## Seguridad / ofensiva

### 1. Reconocimiento de camino

Desde ofensiva, estas herramientas ayudan a inferir:

- cercanía topológica;
- presencia de filtrado;
- saltos de cloud, MPLS o VPN;
- posibles choke points defensivos.

### 2. Limitaciones

No sustituyen captura, routing table ni evidencia de aplicación. Solo muestran la respuesta a una clase específica de probes.

### 3. Uso defensivo

Para Blue Team y sysadmin, son ideales para separar:

- problema de servicio;
- problema de reachability;
- problema de Internet/ISP;
- problema local del host.

## Relacionado

- [[routing-basico-tabla-rutas]]
- [[tcpdump-cheatsheet-y-bpf]]

## Referencias

- `man traceroute`
- `man mtr`
- Microsoft Learn - *pathping*
- RFC 1812 - *Requirements for IP Version 4 Routers*
