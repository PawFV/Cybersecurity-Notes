---
tags: [red-team/recon, windows, dns, active-directory]
created: 2026-04-19
---

# Windows — DNS enum y Zone Transfer

> [!abstract] TL;DR
> Para AXFR apuntás al **DC real**, no al resolver AWS/ISP. `nltest /dsgetdc` te da nombre + IP del DC.

## Encontrar el DC
```powershell
nltest /dsgetdc:<DOMINIO>
```
Devuelve:
- `DC: \\<FQDN>`
- `Address: \\<IP>`  ← esta es la que querés

## Alternativas
```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.<DOMINIO>
$env:USERDNSDOMAIN
$env:LOGONSERVER
nltest /dclist:<DOMINIO>
```

## Zone Transfer (AXFR)

### nslookup interactivo
```powershell
nslookup
> server <IP_DC>
> set type=all
> ls -d <DOMINIO>
```

### PowerShell (a veces limitado para AXFR real)
```powershell
Resolve-DnsName -Name <DOMINIO> -Server <IP_DC> -Type ANY -TcpOnly | Format-List
Resolve-DnsName -Name <DOMINIO> -Server <IP_DC> -Type TXT
```

### Kali (confiable)
```bash
dig @<IP_DC> <DOMINIO> AXFR
```

> [!tip]
> AXFR va por **TCP/53**. Si falla, forzar TCP (`-TcpOnly` / `dig +tcp`).

## Errores comunes
| Error | Causa |
|---|---|
| `Query refused` | Server NO autoritativo del dominio |
| `Non-existent domain` | Dominio no resuelve en ese server |
| `Timeout` | IP incorrecta o puerto filtrado |
| `Format error` | El server no es DNS o no habla AXFR |

## Flags en CTF
Buscar registros **TXT** en el dump → `"THM{...}"` típicamente ahí.