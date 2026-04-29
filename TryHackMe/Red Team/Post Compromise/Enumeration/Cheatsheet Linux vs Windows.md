---
tags:
  - red-team
  - enumeration
  - cheatsheet
  - reference
created: 2026-04-19
source: TryHackMe - Enumeration
related:
  - "[[Windows built-in tools]]"
  - "[[Linux built-in tools]]"
---
# Enum — Cheatsheet Linux vs Windows

> [!abstract] TL;DR
> Misma info, sintaxis distinta. Tabla de equivalencias para pivotear rápido entre OS.

## Equivalencias directas

| Propósito | Linux | Windows |
|---|---|---|
| Hostname | `hostname` | `hostname` / `systeminfo` |
| User actual | `whoami` | `whoami` |
| Users logueados | `who` / `w` | `query user` / `PsLoggedOn` |
| Último login | `last` | `Get-EventLog Security` (PS) |
| IPs | `ip a s` | `ipconfig /all` |
| ARP cache | `arp -a` | `arp -a` |
| Conexiones | `netstat -atupn` | `netstat -abno` |
| Procesos | `ps aux` | `tasklist` / `Get-Process` |
| Users | `cat /etc/passwd` | `net user` |
| Groups | `cat /etc/group` | `net localgroup` |
| Apps instaladas | `rpm -qa` / `dpkg -l` | `wmic product get name` |
| OS info | `cat /etc/os-release` | `systeminfo` |
| Privs actuales | `id` / `sudo -l` | `whoami /priv /groups` |
| Shares de red | `smbclient -L` | `net share` |

## Flujo estándar post-exploit — ambos OS

```
1. quién soy + qué puedo → whoami variants
2. qué sistema es        → OS / versión / patches
3. quién más está        → users, logins recientes
4. qué corre             → processes, services
5. qué conexiones        → netstat, conexiones activas
6. qué red               → IPs, routing, DNS, ARP
7. qué está compartido   → shares, mounts
8. qué secretos          → config files, history, credentials
```

> [!tip] Orden mental
> De adentro hacia afuera: yo → máquina → red. Así priorizás privesc local antes de pivot.