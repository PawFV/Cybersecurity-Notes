---
title: ss, netstat, lsof Procesos vs Puertos
tags: [networking, ss, netstat, lsof, sockets, troubleshooting]
aliases: [Puertos y Procesos, Sockets en Linux, ss vs netstat]
created: 2026-04-23
difficulty: Básico
related:
  - "[[tcp-estados-y-handshake]]"
  - "[[tcpdump-cheatsheet-y-bpf]]"
---

# ss, netstat, lsof Procesos vs Puertos

> [!abstract] TL;DR
> - Un **puerto** no es un proceso: es un identificador de socket usado por el sistema operativo.
> - `ss` muestra sockets y estados; `lsof` ayuda a vincular puertos con procesos y archivos abiertos; `netstat` es más legacy.
> - Para saber por qué una app no escucha o qué proceso ocupa un puerto, estas herramientas son el punto de partida.
> - Diferenciar `LISTEN`, `ESTAB`, `TIME_WAIT` y bind addresses evita muchos diagnósticos errados.

## Concepto

Cuando una aplicación "abre un puerto", en realidad le pide al kernel crear un socket y bindearlo a una IP/puerto. El kernel mantiene ese estado; la app consume o produce datos a través de él.

Por eso conviene separar:

- **proceso**: programa en ejecución;
- **socket**: endpoint de comunicación;
- **puerto**: número asociado al socket;
- **estado**: solo aplica principalmente a TCP.

Errores típicos vienen de mezclar niveles:

- "el puerto está abierto, entonces la app anda";
- "veo `ESTAB`, entonces el usuario está autenticado";
- "escucha en 127.0.0.1, entonces está publicado".

Ninguna de esas inferencias es automática.

## Cómo funciona

### `ss`

`ss` consulta información de sockets del kernel y reemplaza en gran medida a `netstat`.

Estados TCP comunes:

- `LISTEN`: esperando conexiones entrantes.
- `ESTAB`: conexión establecida.
- `TIME_WAIT`: cierre reciente, esperando expiración.
- `SYN-SENT` / `SYN-RECV`: handshake en progreso.

```ascii
Proceso -> socket -> bind(127.0.0.1:8080) -> estado LISTEN
Cliente -> connect() -> handshake TCP -> ESTAB
```

### `lsof`

`lsof` lista archivos abiertos, y en Unix un socket es también un descriptor de archivo. Eso lo hace útil para mapear PID, usuario y puerto.

### `netstat`

Sigue apareciendo en documentación vieja y appliances, pero en Linux moderno `ss` suele ser más rápido y expresivo.

> [!tip] Bind address importa
> `127.0.0.1:8080` y `0.0.0.0:8080` no significan lo mismo. El primero expone localmente; el segundo escucha en todas las interfaces IPv4.

## Comandos / configuración

```bash
# Ver puertos TCP a la escucha
ss -tln

# Ver TCP y UDP con proceso asociado
sudo ss -tulpn

# Ver conexiones establecidas
ss -tan state established

# Filtrar por puerto local
ss -tln '( sport = :443 )'

# Legacy
netstat -tulpn

# Qué proceso usa un puerto
sudo lsof -iTCP:443 -sTCP:LISTEN -nP

# Ver todo lo que tiene abierto un proceso
sudo lsof -p 1234
```

Ejemplo de salida conceptual:

```text
State   Local Address:Port   Peer Address:Port   Process
LISTEN  0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=812,fd=3))
LISTEN  127.0.0.1:5432       0.0.0.0:*           users:(("postgres",pid=991,fd=5))
ESTAB   192.168.56.10:22     192.168.56.50:51822 users:(("sshd",pid=2044,fd=4))
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| `Connection refused` | No hay `LISTEN` en ese puerto/IP. | `ss -tlnp \| grep :443` |
| Funciona localmente pero no desde otra máquina | Servicio bindeado solo a `127.0.0.1`. | `ss -tlnp` y revisar `Local Address` |
| El servicio no arranca porque "el puerto está ocupado" | Otro proceso ya hizo bind. | `sudo lsof -iTCP:8080 -sTCP:LISTEN -nP` |
| Hay muchas conexiones en `TIME_WAIT` | Carga alta o patrón normal de cierre TCP. | `ss -tan state time-wait` |
| La app dice que abrió el puerto pero no aparece | Crash antes de bind, namespace distinto o privilegios insuficientes. | Revisar proceso, logs y namespace |

## Seguridad / ofensiva

### 1. Inventario rápido local

En un host comprometido o bajo investigación, `ss` y `lsof` permiten responder rápido:

- qué está escuchando;
- qué proceso originó una conexión saliente;
- qué puertos inesperados aparecieron;
- si hay túneles o proxys locales.

### 2. Indicadores sospechosos

- listeners en puertos no documentados;
- servicios ligados a `0.0.0.0` sin razón;
- conexiones persistentes hacia IPs externas raras;
- procesos efímeros abriendo sockets breves en loop.

> [!note] Puerto abierto no implica exposición externa
> Puede estar escuchando solo en loopback, dentro de un namespace o detrás de firewall local. Siempre correlacioná con bind address y reachability real.

### 3. Relevancia para pivoting

Muchos tunnels y proxies ofensivos terminan viéndose como:

- listeners locales en puertos altos;
- conexiones salientes a un jump host;
- procesos `ssh`, `socat`, `chisel` o equivalentes.

## Relacionado

- [[tcp-estados-y-handshake]]
- [[tcpdump-cheatsheet-y-bpf]]

## Referencias

- `man ss`
- `man netstat`
- `man lsof`
- iproute2 Documentation
