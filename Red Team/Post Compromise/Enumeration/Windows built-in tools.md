---
tags:
  - red-team
  - enumeration
  - windows
  - post-exploitation
created: 2026-04-19
source: TryHackMe - Enumeration
related:
  - "[[Linux built-in tools]]"
---

# Enumeración Windows — Built-in tools

> [!abstract] TL;DR
> Mismas 4 áreas que Linux, sintaxis distinta. Desde `cmd.exe` podés hacer casi todo sin dropear binarios.

## System

```cmd
systeminfo                               :: OS, version, build, hotfixes, NICs
wmic qfe get Caption,Description         :: Patches instalados detallados
net start                                :: Servicios corriendo
wmic product get name,version,vendor     :: Apps instaladas
```

> [!tip]
> Hotfixes revelan **patch cadence**. Pocos hotfixes = sistema sin mantener = más exploits probables.

## Users

```cmd
whoami                          :: Usuario actual
whoami /priv                    :: Tus privileges (clave para privesc)
whoami /groups                  :: Grupos a los que pertenecés
net user                        :: Todos los users locales
net localgroup                  :: Grupos locales (o net group si es DC)
net localgroup administrators   :: Miembros de admins
net accounts                    :: Password policy (/domain si aplica)
```

> [!warning] whoami /priv
> Si ves `SeImpersonatePrivilege` o `SeAssignPrimaryTokenPrivilege` enabled → buscá **Potato exploits** (JuicyPotato, PrintSpoofer, RoguePotato). Privesc casi garantizado.

## Networking

```cmd
ipconfig               :: Básico
ipconfig /all          :: Con DNS servers, DHCP, MAC
netstat -abno          :: All + Binary + Numeric + PID (más útil)
arp -a                 :: Hosts que recientemente hablaron con el tuyo
```

### netstat flags

| Flag | Significado |
|---|---|
| `-a` | All (listening + established) |
| `-b` | Binario involucrado |
| `-n` | Numérico |
| `-o` | PID |

> [!tip] arp -a
> Revela hosts en el mismo LAN que se comunicaron con el tuyo → mapa parcial de la red **sin generar tráfico**.

## Comandos clave

```cmd
:: Snapshot rápido
systeminfo && whoami /priv && whoami /groups && netstat -abno && arp -a
```

## Por qué no escanear desde afuera
- Firewalls pueden bloquear
- Port scan = ruido → detección
- `netstat` desde dentro = **verdad absoluta, cero tráfico**

## Respuestas del task
- OS Name: **Microsoft Windows Server 2022 Standard**
- OS Version: **10.0.20348**
- Hotfixes: **3**
- Puerto TCP más bajo listening: **22**
- Programa: **sshd.exe**