---
title: SSH Arquitectura y Forwarding
tags: [networking, ssh, tunneling, administration, redteam]
aliases: [SSH Basico, Arquitectura SSH, Tunneles SSH]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[ssh-local-remote-dynamic-forwarding]]"
  - "[[tcp-estados-y-handshake]]"
---

# SSH Arquitectura y Forwarding

> [!abstract] TL;DR
> - **SSH** provee acceso remoto, ejecución de comandos, transferencia de archivos y túneles cifrados sobre TCP.
> - La arquitectura separa **transporte**, **autenticación de usuario** y **canales** lógicos multiplexados.
> - Un solo socket TCP puede cargar shell interactiva, `scp`, `sftp` y port forwarding al mismo tiempo.
> - Entender el modelo de canales evita confundir "sesión SSH" con "shell remota".

## Concepto

SSH nació como reemplazo seguro de Telnet, `rsh`, `rlogin` y FTP administrativo. La idea central no es solo "loguearte remoto", sino establecer un **canal autenticado y cifrado** sobre el que después corrés distintos subsistemas.

En OpenSSH, una conexión típica tiene estas capas:

1. **Transport Layer Protocol**: acuerdo criptográfico, integridad y cifrado.
2. **User Authentication Protocol**: password, clave pública, GSSAPI, etc.
3. **Connection Protocol**: multiplexación de canales como shell, exec, `sftp` o forwards.

Eso explica por qué podés usar una misma conexión para:

- abrir una shell;
- copiar archivos;
- tunelizar un puerto local;
- exponer un puerto remoto;
- montar un proxy SOCKS.

## Cómo funciona

### Flujo de una conexión SSH

```ascii
Cliente SSH                                        Servidor SSH
    | --- TCP 3-way handshake ------------------------> |
    | <---- banner: SSH-2.0-OpenSSH_... -------------- |
    | ---- key exchange / host key verification -----> |
    | <---- parámetros criptográficos ---------------- |
    | ---- autenticación de usuario ------------------> |
    | <---- success / failure ------------------------- |
    | ---- apertura de canales -----------------------> |
    | <---- shell / exec / sftp / forwarding --------- |
```

### Host key vs user key

Hay dos validaciones distintas que conviene no mezclar:

- **Host key**: el cliente verifica a qué servidor se está conectando.
- **User key**: el servidor verifica qué usuario intenta autenticarse.

Si el host key cambia de golpe, puede ser:

- reinstalación legítima;
- rotación planificada;
- DNS apuntando a otro host;
- intento de MITM.

> [!warning] `known_hosts`
> Ignorar cambios en `known_hosts` sin validación externa es una costumbre peligrosa. En entornos comprometidos, ese warning es exactamente la señal que no querés ignorar.

### Canales y multiplexación

SSH no "abre un socket por cada cosa". Abre un TCP principal y adentro crea **canales** identificados. Un canal puede ser:

- `session` para shell o `exec`;
- `direct-tcpip` para forwarding local;
- `forwarded-tcpip` para forwarding remoto;
- subsistema `sftp`.

Eso reduce overhead y permite multiplexación controlada.

```ascii
TCP 22
└── SSH Transport
    ├── Channel 0: shell interactiva
    ├── Channel 1: scp / sftp
    └── Channel 2: port forwarding
```

### Reenvío de puertos

El port forwarding toma un flujo TCP en un extremo y lo encapsula dentro de la sesión SSH hasta el otro extremo.

Ejemplo conceptual:

```ascii
127.0.0.1:8080 -> [Cliente SSH] ==== túnel SSH ==== [Servidor SSH] -> 10.10.20.15:80
```

La sesión SSH no reemplaza routing de red. Solo crea un conducto puntual entre sockets.

## Comandos / configuración

```bash
# Login remoto básico
ssh admin@192.168.56.10

# Autenticación con clave explícita
ssh -i ~/.ssh/id_ed25519 admin@192.168.56.10

# Ejecutar un comando sin shell interactiva
ssh admin@192.168.56.10 "hostnamectl && id"

# Multiplexación de conexiones en cliente OpenSSH
ssh -o ControlMaster=auto -o ControlPersist=5m admin@192.168.56.10

# Transferencia de archivos
scp archivo.txt admin@192.168.56.10:/tmp/
sftp admin@192.168.56.10
```

Configuración de cliente:

```sshconfig
Host jump-lab
    HostName 192.168.56.10
    User admin
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
```

Configuración mínima relevante en servidor:

```text
# /etc/ssh/sshd_config
Port 22
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowTcpForwarding yes
X11Forwarding no
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| `Connection refused` | `sshd` no está escuchando o puerto incorrecto. | `ss -tlnp \| grep :22` |
| Timeout | Firewall, ACL o route ausente. | `tcpdump -n host 192.168.56.10 and port 22` |
| `Permission denied (publickey)` | Clave equivocada, permisos incorrectos o usuario incorrecto. | `ssh -vvv admin@192.168.56.10` |
| Warning de host key cambiado | Reinstalación o posible MITM. | `ssh-keygen -F 192.168.56.10` y validar fingerprint por canal alternativo |
| Forwarding no funciona | `AllowTcpForwarding` deshabilitado o bind/ACL incorrecta. | `ssh -vvv -L 8080:10.10.20.15:80 admin@192.168.56.10` |

> [!note] `-vvv`
> Para SSH, el modo verbose casi siempre te dice en qué etapa exacta falló: DNS, TCP, negociación criptográfica, autenticación o apertura de canal.

## Seguridad / ofensiva

### 1. Pivoting legítimo y ofensivo

SSH es la herramienta más limpia para pivoting cuando tenés acceso a un host intermedio. Con un único puerto permitido (`22/tcp` o uno alternativo) podés alcanzar segmentos internos sin exponer servicios extra.

### 2. Riesgos de configuración

Malas prácticas frecuentes:

- `PasswordAuthentication yes` en Internet;
- reutilización de claves privadas;
- `PermitRootLogin yes`;
- agentes SSH reenviados innecesariamente;
- forwards remotos escuchando en `0.0.0.0`.

> [!danger] Agent forwarding
> `ForwardAgent yes` puede convertir un host comprometido en trampolín para usar tu identidad contra otros sistemas, aunque la clave privada no salga del cliente.

### 3. Enumeración y banner grabbing

Aunque SSH cifra la sesión, el banner inicial y algunos comportamientos permiten fingerprinting:

```bash
nmap -sV -p 22 192.168.56.10
ssh -vvv admin@192.168.56.10
```

### 4. Controles defensivos útiles

- solo claves modernas (`ed25519`, `rsa-sha2-*`);
- MFA o certificados SSH;
- listas de usuarios permitidos;
- rate limiting / fail2ban;
- logging y correlación de port forwards.

## Relacionado

- [[ssh-local-remote-dynamic-forwarding]]
- [[tcp-estados-y-handshake]]

## Referencias

- RFC 4251 - *The Secure Shell (SSH) Protocol Architecture*
- RFC 4252 - *The Secure Shell (SSH) Authentication Protocol*
- RFC 4253 - *The Secure Shell (SSH) Transport Layer Protocol*
- RFC 4254 - *The Secure Shell (SSH) Connection Protocol*
- OpenSSH Manual Pages - `ssh`, `sshd`, `ssh_config`, `sshd_config`
