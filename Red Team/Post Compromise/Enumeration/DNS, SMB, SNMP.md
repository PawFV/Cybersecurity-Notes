---
tags:
  - red-team
  - enumeration
  - dns
  - smb
  - snmp
created: 2026-04-19
source: TryHackMe - Enumeration
related:
  - "[[Windows - DNS enum y zone transfer]]"
  - "[[Windows built-in tools]]"
---
# Enum — DNS, SMB, SNMP

> [!abstract] TL;DR
> Tres protocolos que regalan info lateral: DNS (zone transfer), SMB (shares), SNMP (todo, si tenés el community string).

## DNS — Zone Transfer

Si no está restringido, descargás **todos** los records del servidor:

```bash
dig -t AXFR <DOMAIN> @<DNS_SERVER>
```

Ejemplo:
```bash
dig -t AXFR redteam.thm @10.112.133.220
```

> [!tip]
> Zone transfer revela hosts internos que no figuran en DNS público → mapa de infra casi gratis.

## SMB — Shares

```cmd
net share       :: Desde Windows
```

```bash
smbclient -L //<IP> -N           # Desde Linux sin auth
smbclient -L //<IP> -U <user>    # Con credenciales
```

Buscá shares no-default:
- `C$`, `ADMIN$`, `IPC$` = default
- Cualquier otro nombre = **investigar**

## SNMP

Protocolo diseñado para monitoreo → **mina de info** para atacantes. Si tenés el community string (default suele ser `public` o `private`):

```bash
# Kali / AttackBox
/opt/snmpcheck/snmpcheck.rb <IP> -c <COMMUNITY_STRING>

# Instalación local
git clone https://gitlab.com/kalilinux/packages/snmpcheck.git
cd snmpcheck/
gem install snmp
chmod +x snmpcheck-1.9.rb
```

SNMP puede revelar:
- Usuarios del sistema
- Software instalado
- Procesos corriendo
- Info de hardware
- Tablas de routing
- Shares de red

> [!warning]
> Community strings default (`public`, `private`) en producción = red flag histórica. Los encontrás más de lo que pensás.

## Flujo de ataque típico

```
1. dig AXFR     → descubrir hosts internos
2. nmap -p 445 hosts encontrados
3. smbclient -L → enumerar shares
4. snmpwalk / snmpcheck con community strings comunes
5. Correlacionar info → pivot a siguiente target
```