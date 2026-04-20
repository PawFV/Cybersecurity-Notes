---
tags:
  - red-team
  - enumeration
  - linux
  - post-exploitation
created: 2026-04-19
source: TryHackMe - Enumeration
related:
  - "[[Propósito]]"
  - "[[Enum - Windows built-in tools]]"
---

# Enumeración Linux — Built-in tools

> [!abstract] TL;DR
> 4 áreas: **System, Users, Networking, Services**. Todo con tools que ya están instaladas → cero ruido.

## System

### Distro y versión
```bash
ls /etc/*-release
cat /etc/os-release
hostname
```

### Archivos clave
| Archivo | Contenido | Permisos |
|---|---|---|
| `/etc/passwd` | Users | Todos leen |
| `/etc/group` | Groups | Todos leen |
| `/etc/shadow` | Password hashes | **Solo root** |
| `/var/mail/` | Mails de users | Selectivos |

Si conseguís `/etc/shadow` → crackeás con `hashcat`/`john`.

### Apps instaladas
```bash
ls -lh /usr/bin/
ls -lh /sbin/

# RPM-based (RHEL, CentOS, Fedora)
rpm -qa

# Debian-based (Ubuntu, Debian, Kali)
dpkg -l
```

## Users

```bash
who          # Quién está logueado ahora + desde qué IP
w            # Quién + qué está haciendo
whoami       # Tu UID efectivo
id           # UID, GID, grupos
last         # Historial de logins (incluye duración de sesión)
sudo -l      # Qué comandos podés correr con sudo
```

> [!tip]
> `sudo -l` es oro puro para privesc. Si ves NOPASSWD en algo → casi siempre escalable.

## Networking

### IPs y DNS
```bash
ip a s               # IPs (moderno)
ifconfig -a          # Legacy
cat /etc/resolv.conf # DNS servers
```

### Conexiones — netstat
```bash
netstat -plt    # Programas LISTENING en TCP
netstat -atupn  # TODO: TCP+UDP, listening+established, con PID, numérico
```

| Flag | Significado |
|---|---|
| `-a` | All (listening + non-listening) |
| `-l` | Solo listening |
| `-n` | Numérico (no resuelve DNS/ports) |
| `-t` | TCP |
| `-u` | UDP |
| `-p` | PID + nombre del programa |

### Conexiones — lsof
```bash
sudo lsof -i         # Todas las conexiones de red
sudo lsof -i :25     # Solo puerto 25
```

> [!warning]
> Corré netstat/lsof como root o con sudo para ver **todos** los PIDs.

### Por qué no usar nmap
Nmap genera muchos paquetes → triggerea IDS/IPS. Firewalls pueden dropear → resultado incompleto. `netstat` desde dentro = **zero noise, ground truth**.

## Running services

```bash
ps -e           # Todos los procesos
ps -ef          # Full format
ps aux          # BSD syntax, con info de usuario
ps axf          # Process tree (forest)
ps -ef | grep peter  # Filtrar por user
```

## Comandos clave

```bash
# Snapshot rápido de todo
hostname && whoami && id && ip a s && sudo netstat -atupn && ps aux
```

## Respuestas del task
- Distro: **CentOS Linux** / versión: **7.9.2009**
- Último user: **randa**
- Puerto TCP más alto listening: **2049** (program: **rpc.mountd**)
- Script en background: **THMbackup.sh**

> [!note]
> Valores concretos dependen de la VM del lab. Verificá con los comandos en tu sesión.