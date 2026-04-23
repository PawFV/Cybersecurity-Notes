---
title: iptables, conntrack y filtrado stateful
tags: [networking, linux, firewall, iptables, conntrack]
aliases: [iptables stateful, conntrack con iptables]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[iptables-tablas-chains-targets]]"
  - "[[nftables-migracion-desde-iptables]]"
  - "[[nat-snat-dnat-masquerade]]"
---

# iptables, conntrack y filtrado stateful

> [!abstract] TL;DR
> - **conntrack** mantiene estado de flujos en el kernel para que el firewall pueda decidir con memoria y no paquete por paquete en aislamiento.
> - La regla más clásica en Linux es aceptar `ESTABLISHED,RELATED` y después abrir solo `NEW` estrictamente necesario.
> - `-m state` es legado; en 2026 preferís `-m conntrack --ctstate ...`.
> - Si conntrack se rompe, se llena o ve tráfico asimétrico, el síntoma no siempre es "bloqueo"; muchas veces ves timeouts, resets extraños o NAT inconsistente.

## Concepto

Un firewall **stateless** mira un paquete como una foto suelta. Un firewall **stateful** mira la película.

Conntrack hace eso: guarda una entrada por flujo y recuerda datos como:

- origen y destino,
- puertos,
- protocolo,
- dirección original y traducida si hubo NAT,
- estado del intercambio.

En TCP el concepto es intuitivo porque existe handshake. En UDP e ICMP también hay seguimiento, aunque más heurístico: el kernel observa ida, vuelta, timeouts y asociaciones relacionadas.

Ejemplo práctico:

- Permitís `NEW` hacia `tcp/443` saliente.
- Cuando vuelve la respuesta del servidor desde `203.0.113.20:443` a tu puerto efímero, no necesitás otra regla explícita.
- Conntrack ya sabe que ese tráfico pertenece a una sesión existente y lo clasifica como `ESTABLISHED`.

## Cómo funciona

### Estados más usados

Los estados de `conntrack` que vas a usar todo el tiempo son:

- **`NEW`**: primer paquete de una conexión o flujo observado.
- **`ESTABLISHED`**: ya hubo tráfico en ambos sentidos y la sesión está viva.
- **`RELATED`**: flujo nuevo, pero derivado de otro ya permitido.
  - ejemplo clásico: canal de datos FTP activo/pasivo con helper,
  - o ciertos errores ICMP asociados a una conexión ya existente.
- **`INVALID`**: paquete que no puede asociarse correctamente a una entrada válida.
- **`UNTRACKED`**: tráfico excluido de conntrack, típicamente con `NOTRACK` en tabla `raw`.

### Dónde engancha conntrack

Simplificado:

```ascii
Paquete entra
   |
   v
[raw PREROUTING] ----> puede marcarse NOTRACK
   |
   v
[conntrack lookup/create]
   |
   v
[mangle/nat/filter]
   |
   v
Decisión final
```

En tráfico local generado por el host ocurre algo equivalente en `OUTPUT`.

### Regla stateful clásica

```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

La lógica es:

1. aceptá retorno y tráfico asociado,
2. descartá basura,
3. permití sesiones nuevas solo donde corresponde.

### NAT y conntrack

Conntrack y NAT están íntimamente ligados. Cuando hacés SNAT o DNAT, el kernel mantiene:

- tupla original,
- tupla traducida,
- asociación entre ambas.

Por eso una respuesta a un `DNAT` puede volver correctamente al cliente original sin que vos escribas reglas espejo manuales para cada paquete de vuelta.

> [!tip]
> Si hacés troubleshooting de NAT, pensá siempre en tres planos al mismo tiempo: ruta, regla de traducción y entrada en conntrack. Mirar solo `iptables -t nat -L` te deja ciego a la mitad del problema.

## Comandos / configuración

Reglas de ejemplo para un host con política restrictiva:

```bash
# Política base
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Tráfico local y retorno
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Descartar inválidos temprano
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Permitir SSH solo desde red de administración
sudo iptables -A INPUT -p tcp -s 10.30.40.0/24 --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# Permitir DNS saliente y su retorno
sudo iptables -A OUTPUT -p udp --dport 53 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p udp --sport 53 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Inspección de la tabla de conntrack:

```bash
# Requiere conntrack-tools
sudo conntrack -L
sudo conntrack -L -p tcp
sudo conntrack -L -s 10.30.40.15

# Eventos en tiempo real
sudo conntrack -E
```

Parámetros útiles del kernel:

```bash
# Tamaño máximo de entradas
sysctl net.netfilter.nf_conntrack_max

# Contador actual de entradas
sysctl net.netfilter.nf_conntrack_count

# Timeouts relevantes
sysctl net.netfilter.nf_conntrack_tcp_timeout_established
sysctl net.netfilter.nf_conntrack_udp_timeout
sysctl net.netfilter.nf_conntrack_udp_timeout_stream
```

Exclusión selectiva de conntrack:

```bash
# Evitar tracking para tráfico específico en raw
sudo iptables -t raw -A PREROUTING -p udp --dport 514 -j NOTRACK
sudo iptables -t raw -A OUTPUT -p udp --sport 514 -j NOTRACK
```

> [!warning]
> Sacar tráfico de conntrack puede mejorar rendimiento en casos puntuales, pero te deja sin estado, sin parte del NAT y sin visibilidad útil. Hacelo solo cuando sabés exactamente por qué.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| La sesión inicia y después se corta aleatoriamente | Tabla conntrack llena o timeout agresivo | `sysctl net.netfilter.nf_conntrack_count`, `sysctl net.netfilter.nf_conntrack_max` |
| DNAT funciona a veces y a veces no | Routing asimétrico o entradas viejas/stale | `sudo conntrack -L`, `ip route`, `tcpdump -ni any host 10.10.20.15` |
| Mucho tráfico marcado `INVALID` | Fragmentación, offloads, paquetes fuera de secuencia o asimetría | `sudo iptables -L -n -v`, `ethtool -k eth0`, `tcpdump -ni eth0` |
| UDP "muere" rápido pese a estar permitido | Timeout UDP demasiado bajo para el patrón real | `sysctl net.netfilter.nf_conntrack_udp_timeout_stream` |
| El firewall deja pasar retorno TCP pero rompe FTP/SIP viejos | Faltan helpers o están deshabilitados a propósito | `lsmod | rg nf_conntrack`, `sudo conntrack -L` |

> [!note]
> `RELATED` no es magia universal. Funciona bien para protocolos y mensajes que el kernel puede asociar razonablemente. Si el protocolo es raro, encapsulado o cifrado de forma opaca, no esperes milagros.

## Seguridad / ofensiva

### 1. `ESTABLISHED,RELATED` es correcto, pero no reemplaza criterio

Muchos admins lo tratan como amuleto y no como regla controlada. Si abrís demasiado `NEW`, el retorno `ESTABLISHED` solo acelera la exposición.

### 2. Helpers de aplicación

Históricamente, helpers como FTP o SIP ampliaban muchísimo la superficie de inspección del kernel. En entornos modernos:

- se prefieren protocolos más simples,
- se evita helper automático indiscriminado,
- se controla explícitamente cuándo cargar o usar esos módulos.

Menos parsing en kernel, menos sorpresas.

### 3. Exhaustión de conntrack

Un ataque de flood no siempre busca tirar el servicio final. A veces apunta a llenar la tabla de seguimiento:

- SYN floods,
- flujos UDP efímeros masivos,
- scans distribuidos con mucha cardinalidad de 5-tuplas.

Cuando la tabla se satura, el daño colateral pega sobre tráfico legítimo.

### 4. Rutas asimétricas y evasión

Si el ida pasa por un firewall stateful y la vuelta no, o viceversa, el estado se rompe. Desde Red Team esto es útil para identificar:

- segmentaciones mal diseñadas,
- balanceos raros,
- NAT parcial,
- appliances que filtran en un solo sentido.

Desde Blue Team, asimetría + statefulness = tickets eternos.

### 5. Legado vs presente

En 2026, seguir escribiendo policy nueva en `iptables` suele ser deuda técnica. El modelo stateful sigue vigente, pero la implementación recomendada para reglas nuevas es **nftables**. Entender conntrack sigue siendo obligatorio porque `nft` también se apoya en él.

> [!danger]
> Borrar entradas de conntrack en producción para "probar rápido" puede tumbar sesiones legítimas, VPNs, NATs activos y tráfico encapsulado. Si vas a purgar, delimitá por origen, destino o protocolo, no a ciegas.

## Relacionado

- [[iptables-tablas-chains-targets]] (Base estructural de tablas, chains y targets)
- [[nftables-migracion-desde-iptables]] (Equivalente moderno)
- [[nat-snat-dnat-masquerade]] (NAT y su interacción con estado)

## Referencias

- `man iptables-extensions`
- `man conntrack`
- `man conntrackd`
- Linux kernel documentation: [https://docs.kernel.org/networking/nf_conntrack-sysctl.html](https://docs.kernel.org/networking/nf_conntrack-sysctl.html)
- Netfilter project documentation: [https://www.netfilter.org/documentation/](https://www.netfilter.org/documentation/)
---
title: iptables, conntrack y filtrado stateful
tags: [networking, linux, firewall, iptables, conntrack]
aliases: [iptables stateful, conntrack con iptables]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[iptables-tablas-chains-targets]]"
  - "[[nftables-migracion-desde-iptables]]"
  - "[[nat-snat-dnat-masquerade]]"
---

# iptables, conntrack y filtrado stateful

> [!abstract] TL;DR
> - **conntrack** mantiene estado de flujos en el kernel para que el firewall pueda decidir con memoria y no paquete por paquete en aislamiento.
> - La regla más clásica en Linux es aceptar `ESTABLISHED,RELATED` y después abrir solo `NEW` estrictamente necesario.
> - `-m state` es legado; en 2026 preferís `-m conntrack --ctstate ...`.
> - Si conntrack se rompe, se llena o ve tráfico asimétrico, el síntoma no siempre es "bloqueo"; muchas veces ves timeouts, resets extraños o NAT inconsistente.

## Concepto

Un firewall **stateless** mira un paquete como una foto suelta. Un firewall **stateful** mira la película.

Conntrack hace eso: guarda una entrada por flujo y recuerda datos como:

- origen y destino,
- puertos,
- protocolo,
- dirección original y traducida si hubo NAT,
- estado del intercambio.

En TCP el concepto es intuitivo porque existe handshake. En UDP e ICMP también hay seguimiento, aunque más heurístico: el kernel observa ida, vuelta, timeouts y asociaciones relacionadas.

Ejemplo práctico:

- Permitís `NEW` hacia `tcp/443` saliente.
- Cuando vuelve la respuesta del servidor desde `203.0.113.20:443` a tu puerto efímero, no necesitás otra regla explícita.
- Conntrack ya sabe que ese tráfico pertenece a una sesión existente y lo clasifica como `ESTABLISHED`.

## Cómo funciona

### Estados más usados

Los estados de `conntrack` que vas a usar todo el tiempo son:

- **`NEW`**: primer paquete de una conexión o flujo observado.
- **`ESTABLISHED`**: ya hubo tráfico en ambos sentidos y la sesión está viva.
- **`RELATED`**: flujo nuevo, pero derivado de otro ya permitido.
  - ejemplo clásico: canal de datos FTP activo/pasivo con helper,
  - o ciertos errores ICMP asociados a una conexión ya existente.
- **`INVALID`**: paquete que no puede asociarse correctamente a una entrada válida.
- **`UNTRACKED`**: tráfico excluido de conntrack, típicamente con `NOTRACK` en tabla `raw`.

### Dónde engancha conntrack

Simplificado:

```ascii
Paquete entra
   |
   v
[raw PREROUTING] ----> puede marcarse NOTRACK
   |
   v
[conntrack lookup/create]
   |
   v
[mangle/nat/filter]
   |
   v
Decisión final
```

En tráfico local generado por el host ocurre algo equivalente en `OUTPUT`.

### Regla stateful clásica

```bash
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

La lógica es:

1. aceptá retorno y tráfico asociado,
2. descartá basura,
3. permití sesiones nuevas solo donde corresponde.

### NAT y conntrack

Conntrack y NAT están íntimamente ligados. Cuando hacés SNAT o DNAT, el kernel mantiene:

- tupla original,
- tupla traducida,
- asociación entre ambas.

Por eso una respuesta a un `DNAT` puede volver correctamente al cliente original sin que vos escribas reglas espejo manuales para cada paquete de vuelta.

> [!tip]
> Si hacés troubleshooting de NAT, pensá siempre en tres planos al mismo tiempo: ruta, regla de traducción y entrada en conntrack. Mirar solo `iptables -t nat -L` te deja ciego a la mitad del problema.

## Comandos / configuración

Reglas de ejemplo para un host con política restrictiva:

```bash
# Política base
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Tráfico local y retorno
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Descartar inválidos temprano
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Permitir SSH solo desde red de administración
sudo iptables -A INPUT -p tcp -s 10.30.40.0/24 --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# Permitir DNS saliente y su retorno
sudo iptables -A OUTPUT -p udp --dport 53 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p udp --sport 53 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Inspección de la tabla de conntrack:

```bash
# Requiere conntrack-tools
sudo conntrack -L
sudo conntrack -L -p tcp
sudo conntrack -L -s 10.30.40.15

# Eventos en tiempo real
sudo conntrack -E
```

Parámetros útiles del kernel:

```bash
# Tamaño máximo de entradas
sysctl net.netfilter.nf_conntrack_max

# Contador actual de entradas
sysctl net.netfilter.nf_conntrack_count

# Timeouts relevantes
sysctl net.netfilter.nf_conntrack_tcp_timeout_established
sysctl net.netfilter.nf_conntrack_udp_timeout
sysctl net.netfilter.nf_conntrack_udp_timeout_stream
```

Exclusión selectiva de conntrack:

```bash
# Evitar tracking para tráfico específico en raw
sudo iptables -t raw -A PREROUTING -p udp --dport 514 -j NOTRACK
sudo iptables -t raw -A OUTPUT -p udp --sport 514 -j NOTRACK
```

> [!warning]
> Sacar tráfico de conntrack puede mejorar rendimiento en casos puntuales, pero te deja sin estado, sin parte del NAT y sin visibilidad útil. Hacelo solo cuando sabés exactamente por qué.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| La sesión inicia y después se corta aleatoriamente | Tabla conntrack llena o timeout agresivo | `sysctl net.netfilter.nf_conntrack_count`, `sysctl net.netfilter.nf_conntrack_max` |
| DNAT funciona a veces y a veces no | Routing asimétrico o entradas viejas/stale | `sudo conntrack -L`, `ip route`, `tcpdump -ni any host 10.10.20.15` |
| Mucho tráfico marcado `INVALID` | Fragmentación, offloads, paquetes fuera de secuencia o asimetría | `sudo iptables -L -n -v`, `ethtool -k eth0`, `tcpdump -ni eth0` |
| UDP "muere" rápido pese a estar permitido | Timeout UDP demasiado bajo para el patrón real | `sysctl net.netfilter.nf_conntrack_udp_timeout_stream` |
| El firewall deja pasar retorno TCP pero rompe FTP/SIP viejos | Faltan helpers o están deshabilitados a propósito | `lsmod | rg nf_conntrack`, `sudo conntrack -L` |

> [!note]
> `RELATED` no es magia universal. Funciona bien para protocolos y mensajes que el kernel puede asociar razonablemente. Si el protocolo es raro, encapsulado o cifrado de forma opaca, no esperes milagros.

## Seguridad / ofensiva

### 1. `ESTABLISHED,RELATED` es correcto, pero no reemplaza criterio

Muchos admins lo tratan como amuleto y no como regla controlada. Si abrís demasiado `NEW`, el retorno `ESTABLISHED` solo acelera la exposición.

### 2. Helpers de aplicación

Históricamente, helpers como FTP o SIP ampliaban muchísimo la superficie de inspección del kernel. En entornos modernos:

- se prefieren protocolos más simples,
- se evita helper automático indiscriminado,
- se controla explícitamente cuándo cargar o usar esos módulos.

Menos parsing en kernel, menos sorpresas.

### 3. Exhaustión de conntrack

Un ataque de flood no siempre busca tirar el servicio final. A veces apunta a llenar la tabla de seguimiento:

- SYN floods,
- flujos UDP efímeros masivos,
- scans distribuidos con mucha cardinalidad de 5-tuplas.

Cuando la tabla se satura, el daño colateral pega sobre tráfico legítimo.

### 4. Rutas asimétricas y evasión

Si el ida pasa por un firewall stateful y la vuelta no, o viceversa, el estado se rompe. Desde Red Team esto es útil para identificar:

- segmentaciones mal diseñadas,
- balanceos raros,
- NAT parcial,
- appliances que filtran en un solo sentido.

Desde Blue Team, asimetría + statefulness = tickets eternos.

### 5. Legado vs presente

En 2026, seguir escribiendo policy nueva en `iptables` suele ser deuda técnica. El modelo stateful sigue vigente, pero la implementación recomendada para reglas nuevas es **nftables**. Entender conntrack sigue siendo obligatorio porque `nft` también se apoya en él.

> [!danger]
> Borrar entradas de conntrack en producción para "probar rápido" puede tumbar sesiones legítimas, VPNs, NATs activos y tráfico encapsulado. Si vas a purgar, delimitá por origen, destino o protocolo, no a ciegas.

## Relacionado

- [[iptables-tablas-chains-targets]] (Base estructural de tablas, chains y targets)
- [[nftables-migracion-desde-iptables]] (Equivalente moderno)
- [[nat-snat-dnat-masquerade]] (NAT y su interacción con estado)

## Referencias

- `man iptables-extensions`
- `man conntrack`
- `man conntrackd`
- Linux kernel documentation: [https://docs.kernel.org/networking/nf_conntrack-sysctl.html](https://docs.kernel.org/networking/nf_conntrack-sysctl.html)
- Netfilter project documentation: [https://www.netfilter.org/documentation/](https://www.netfilter.org/documentation/)
