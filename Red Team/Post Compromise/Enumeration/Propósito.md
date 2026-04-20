---
tags:
  - red-team
  - enumeration
  - post-exploitation
  - fundamentals
created: 2026-04-19
source: TryHackMe - Enumeration
related:
  - "[[DNS, SMB, SNMP]]"
  - "[[Windows built-in tools]]"
  - "[[Linux built-in tools]]"
---

> [!abstract] TL;DR
> Post-exploit, estás en una "habitación oscura". Enumeración = encender la luz. Objetivo: info para **pivotar** a otros sistemas o **saquear** el actual.

## Contexto
Ya accediste al sistema (shell básico o con privesc). Aunque seas unprivileged, muchas tools todavía te dan info útil sin hacer ruido — usás binarios que ya están en el sistema, nada sospechoso para el SOC.

## Info que buscás

- Users y groups
- Hostnames
- Routing tables
- Network shares
- Servicios corriendo
- Apps + banners
- Configuraciones
- Service settings + audit configs
- Detalles SNMP / DNS
- **Credenciales** (browsers, apps, archivos)

## Lo que típicamente encontrás

- `passwords.txt` / `passwords.xlsx` en escritorios (clásico)
- SSH keys privadas → acceso a otros servers donde la pública esté instalada
- Source code con secrets embebidos (API keys, DB creds)
- Archivos sensibles en carpetas de user

> [!tip]
> Cambiar de shell es trivial. Desde `cmd.exe` → `powershell.exe`. Desde `sh` → `bash`. Cada shell tiene capacidades distintas.

## Recursos relacionados
- [[WinPEAS]] — privesc automation Windows
- [[LinPEAS]] — privesc automation Linux