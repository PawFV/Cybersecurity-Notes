---
title: SSH Local, Remote y Dynamic Forwarding
tags: [networking, ssh, forwarding, socks, pivoting]
aliases: [SSH Port Forwarding, SSH -L -R -D, Tunneles SSH]
created: 2026-04-23
difficulty: Avanzado
related:
  - "[[ssh-arquitectura-y-forwarding]]"
  - "[[smb-cifs-y-shares-windows]]"
---

# SSH Local, Remote y Dynamic Forwarding

> [!abstract] TL;DR
> - **Local forwarding (`-L`)** expone localmente un servicio remoto o interno.
> - **Remote forwarding (`-R`)** expone en el servidor SSH un servicio que vive del lado cliente.
> - **Dynamic forwarding (`-D`)** crea un proxy SOCKS para tunelizar múltiples destinos vía SSH.
> - Entender desde qué lado escucha cada socket evita errores operativos y exposición accidental.

## Concepto

Port forwarding en SSH es una forma de **transportar TCP dentro de TCP** usando la sesión SSH como conducto. No reemplaza VPN, no propaga broadcast, no reescribe rutas automáticamente y no "descubre" redes. Lo que hace es más quirúrgico: conecta sockets concretos.

La confusión típica aparece porque los nombres `local` y `remote` no hablan del destino final, sino de **dónde se abre el socket que escucha**.

- `-L`: el puerto escucha en tu máquina cliente.
- `-R`: el puerto escucha del lado servidor SSH.
- `-D`: el puerto escucha en tu máquina y habla SOCKS.

## Cómo funciona

### 1. Local forwarding (`-L`)

Caso clásico: desde tu notebook querés llegar a un web interno accesible solo desde un jump host.

```ascii
Browser -> 127.0.0.1:8080
              |
              v
        [cliente SSH] ==== túnel ==== [jump host] -> 10.20.30.40:80
```

Comando:

```bash
ssh -L 8080:10.20.30.40:80 admin@192.168.56.10
```

Esto significa:

- abrir `127.0.0.1:8080` en tu cliente;
- encapsular conexiones dentro de SSH;
- pedirle al servidor SSH que llegue a `10.20.30.40:80`.

### 2. Remote forwarding (`-R`)

Caso inverso: querés exponer en el servidor SSH algo que existe de tu lado cliente.

```ascii
usuario en servidor -> 127.0.0.1:8443
                           |
                           v
                    [servidor SSH] ==== túnel ==== [cliente SSH] -> 127.0.0.1:443
```

Comando:

```bash
ssh -R 8443:127.0.0.1:443 admin@192.168.56.10
```

Muy útil para:

- publicar temporalmente un servicio local;
- recibir callbacks;
- exponer herramientas en laboratorios controlados.

> [!warning] `GatewayPorts`
> Si el servidor permite `GatewayPorts yes`, un `-R` puede escuchar en interfaces no loopback. Eso cambia mucho el riesgo: deja de ser "visible solo en el servidor" y puede volverse accesible desde terceros.

### 3. Dynamic forwarding (`-D`)

`-D` crea un proxy SOCKS. En vez de fijar un único destino, la aplicación cliente negocia a qué IP/puerto quiere conectarse.

```ascii
Browser/Proxychains -> 127.0.0.1:1080 (SOCKS)
                           |
                           v
                    [cliente SSH] ==== túnel ==== [servidor SSH] -> destinos varios
```

Comando:

```bash
ssh -D 1080 admin@192.168.56.10
```

Ideal para navegación selectiva, tooling con proxy SOCKS y pivoting liviano.

### 4. Bind addresses

Un detalle operativo importante:

- `-L 8080:host:80` suele bindear en `127.0.0.1`.
- `-L 0.0.0.0:8080:host:80` expone en todas las interfaces locales.
- `-R 8443:127.0.0.1:443` depende de políticas de `sshd`.

> [!danger] Exposición involuntaria
> Un bind en `0.0.0.0` transforma una herramienta de administración en un servicio publicado. Si tu host tiene varias interfaces o VPNs, podrías exponer datos sensibles sin darte cuenta.

## Comandos / configuración

```bash
# Local forwarding: publicar localmente un web interno
ssh -N -L 8080:10.20.30.40:80 admin@192.168.56.10

# Remote forwarding: exponer un web local en el servidor remoto
ssh -N -R 8443:127.0.0.1:443 admin@192.168.56.10

# Dynamic forwarding: levantar proxy SOCKS5 local
ssh -N -D 1080 admin@192.168.56.10

# Encadenar múltiples forwards
ssh -N \
  -L 8080:10.20.30.40:80 \
  -L 3389:10.20.30.50:3389 \
  -D 1080 \
  admin@192.168.56.10

# Saltar por un bastion
ssh -J ops@198.51.100.20 admin@10.20.30.10
```

Cliente OpenSSH:

```sshconfig
Host pivot-lab
    HostName 192.168.56.10
    User admin
    DynamicForward 1080
    LocalForward 8080 10.20.30.40:80
```

Server side relevante:

```text
# /etc/ssh/sshd_config
AllowTcpForwarding yes
GatewayPorts no
PermitOpen any
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| `bind: Address already in use` | El puerto local/remoto ya está ocupado. | `ss -tlnp \| grep :8080` |
| El túnel abre pero no conecta al destino | El servidor SSH no tiene reachability al host destino. | Desde el jump host: `nc -vz 10.20.30.40 80` |
| `remote port forwarding failed` | `AllowTcpForwarding`, `PermitOpen` o políticas del server lo impiden. | `ssh -vvv -R 8443:127.0.0.1:443 admin@192.168.56.10` |
| SOCKS levanta pero el tráfico no pasa | La app no está usando el proxy o resuelve DNS fuera del túnel. | Probar con `curl --socks5-hostname 127.0.0.1:1080 http://example.com` |
| El servicio quedó expuesto a terceros | Bind en `0.0.0.0` o `GatewayPorts yes`. | `ss -tlnp` y revisar dirección de escucha |

## Seguridad / ofensiva

### 1. Pivoting de bajo ruido

Para Red Team, SSH forwarding suele generar menos fricción que desplegar agentes adicionales o VPNs completas. Aprovecha un canal ya permitido y confiable desde la perspectiva del entorno.

### 2. Riesgos defensivos

Para Blue Team, el problema es la **opacidad funcional**: un `ssh` aparentemente legítimo puede transportar RDP, SMB, bases de datos o exfiltración HTTP por dentro.

Controles útiles:

- restringir `AllowTcpForwarding`;
- registrar destinos permitidos con `PermitOpen`;
- monitorear procesos que bindean puertos locales anómalos;
- correlacionar sesiones SSH largas sin shell.

### 3. Casos típicos ofensivos

- acceder a un panel web interno;
- usar `proxychains` sobre `-D`;
- exponer un listener local mediante `-R`;
- encadenar bastions con `ProxyJump`.

```bash
proxychains4 nmap -sT -Pn 10.20.30.40 -p 445
```

> [!note] Nmap y SOCKS
> Sobre SOCKS no todo scan funciona igual. `-sS` requiere raw sockets; para proxys lo normal es caer en `-sT`.

## Relacionado

- [[ssh-arquitectura-y-forwarding]]
- [[smb-cifs-y-shares-windows]]

## Referencias

- RFC 4254 - *The Secure Shell (SSH) Connection Protocol*
- OpenSSH Manual Pages - `ssh`, `ssh_config`, `sshd_config`
- OpenSSH Documentation - *PROTOCOL*
