---
tags:
  - red-team/recon
  - windows
  - enumeration
  - evasion
created: 2026-04-19
---

# Windows — Enumeración AV/EDR

> [!abstract] TL;DR
> `SecurityCenter2` solo existe en Windows Desktop. Servers → `Get-MpComputerStatus` + enum de servicios/procesos.

## Identificar OS primero
```powershell
(Get-CimInstance Win32_OperatingSystem).Caption
(Get-CimInstance Win32_OperatingSystem).ProductType
```
ProductType: `1`=Workstation, `2`=DC, `3`=Server

## Namespaces WMI según OS
| OS               | Funciona                                                                      |
| ---------------- | ----------------------------------------------------------------------------- |
| Win10/11 Desktop | `Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntivirusProduct` |
| Server           | ❌ No existe — por diseño                                                      |

### Errores típicos
- `Invalid namespace` → no existe (Server)
- `Access denied` → existe pero sin permisos (`root/SECURITY` necesita SYSTEM)
- `Invalid class` → namespace ok pero clase incorrecta

### Listar namespaces disponibles
```powershell
Get-CimInstance -Namespace root -ClassName __Namespace | Select Name
```

## Enum universal (funciona en Server)

### Defender
```powershell
Get-MpComputerStatus | Select AMServiceEnabled, AntivirusEnabled, RealTimeProtectionEnabled, AMProductVersion
Get-MpPreference
```

### Servicios AV/EDR conocidos
```powershell
Get-Service | Where-Object {$_.Name -match "defend|mcafee|symantec|sophos|crowdstrike|sentinel|eset|kaspersky|trend|carbon|cylance|cybereason"} | Format-Table Name, Status
```

### Procesos sospechosos
```powershell
Get-Process | Where-Object {$_.ProcessName -match "MsMpEng|mfemms|ccSvcHst|SophosFS|CSFalcon|SentinelAgent|cyserver|ekrn"} | Select ProcessName, Path
```