---
tags:
  - red-team
  - enumeration
  - windows
  - tools
created: 2026-04-19
source: TryHackMe - Enumeration
related:
  - "[[Windows built-in tools]]"
---

# Enum Windows — Tools adicionales (no built-in)

> [!abstract] TL;DR
> Cuando los built-in no alcanzan: Sysinternals (MS oficial), Process Hacker (GUI power), Seatbelt (red team-focused).

## Sysinternals Suite (Microsoft oficial)

Paquete de utilidades CLI + GUI firmadas por Microsoft → **trust alto, LOLBAS friendly**.

| Utility              | Función                                       |
| -------------------- | --------------------------------------------- |
| **Process Explorer** | Procesos + archivos abiertos + registry keys  |
| **Process Monitor**  | Monitoreo real-time de FS, procesos, registry |
| **PsList**           | Info de procesos                              |
| **PsLoggedOn**       | Users logueados (local + remoto)              |
| **PsExec**           | Ejecución remota (clásico lateral movement)   |

> [!tip]
> Muchas herramientas Sysinternals están **whitelisted** en entornos corporativos porque son de Microsoft. Úsalas en tu favor.

## Process Hacker

GUI avanzado, alternativa a Task Manager con esteroides:
- Procesos + conexiones de red activas
- Utilización de CPU, RAM, disk, network
- Inyección de DLLs, manipulación de handles
- Strings de memoria

> [!warning]
> En máquinas target, Process Hacker puede ser detectado por EDR como sospechoso. Úsalo más para análisis en tu Kali/VM que en víctimas.

## GhostPack Seatbelt

- Escrito en **C#** (GhostPack collection)
- Orientado a **red team**
- Situational awareness en un solo binario
- NO release oficial en binario → compilar con Visual Studio

Info que recolecta:
- AV/EDR instalados
- UAC config
- Wireless profiles con passwords
- Saved RDP sessions
- Chrome/Firefox bookmarks + history
- Recent files
- PowerShell history

```powershell
Seatbelt.exe -group=all
Seatbelt.exe -group=system
Seatbelt.exe -group=user
```

> [!tip]
> Seatbelt replica lo que haría un operator chequeando 30 cosas a mano. Un solo ejecutable = situational awareness completo.

## Respuesta del task
- Utility que muestra users logueados: **PsLoggedOn**