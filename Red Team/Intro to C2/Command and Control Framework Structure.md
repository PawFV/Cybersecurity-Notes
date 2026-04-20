---
tags: [red-team, c2, post-exploitation, evasion, fundamentals]
created: 2026-04-19
source: TryHackMe - Intro to C2
related:
  - "[[Red Team - CONOPS]]"
  - "[[CTI - Frameworks y workflow Red Team]]"
  - "[[Cyber Kill Chain]]"
  - "[[Post Exploitation Basics]]"
  - "[[Weaponization - Payload Formats]]"
---

# C2 Framework — Fundamentos

> [!abstract] TL;DR
> C2 = Netcat con esteroides. Lo que lo diferencia de un reverse shell genérico es **session management + post-exploitation modules + evasión built-in**. Server centraliza agents; agents beaconean; profiles/fronting esconden tráfico.

## Componentes core

| Componente | Función |
|---|---|
| **C2 Server** | Hub donde los agents callbackean. Espera instrucciones del operator. |
| **Agent / Payload** | Programa en la víctima que beaconea al listener. Pseudo-commands (download, upload, etc.). Configurable. |
| **Listener** | App en el C2 que escucha callbacks por puerto/protocolo (DNS, HTTP, HTTPS). |
| **Beacon** | El acto de que el agent llame al listener. |

## Obfuscación de callbacks

### Sleep timer
Intervalo fijo entre beacons. **Detectable** — genera patrón regular en NetFlow/firewall.

```
TCP/443 cada 5s exactos → red flag inmediata para SOC
```

### Jitter
Variación random al sleep → rompe patrón, se mimetiza con tráfico humano.

```python
import random
sleep = 60
jitter = random.randint(-30, 30)
sleep = sleep + jitter
```

> [!tip] Jitter avanzado
> En C2 maduros: bounds superiores/inferiores, porcentajes del último sleep, **file jitter** (ruido en tamaño de payloads), junk data para ofuscar tamaños reales.

## Payload types

### Stageless
Payload completo en un solo archivo. Ejecuta → beaconea directo.

```
[Dropper] → [C2 beacon start]
```

**Pros:** simple. **Contras:** más grande, más fácil de detectar por AV.

### Staged
Dropper mínimo descarga el resto desde el C2.

```
[Dropper] → [call C2 for stage 2] → [load stage 2 in memory] → [C2 beacon start]
```

**Pros:** dropper pequeño, más fácil de ofuscar para evasión AV. **Contras:** requiere callback inicial → más ruido de red.

> [!tip]
> Staged es preferido cuando el target tiene AV/EDR agresivo. Stage 2 se carga **en memoria** → no toca disco.

## Payload formats
No solo `.exe` (PE). Opciones comunes:
- PowerShell scripts (pueden embedar C# compilado con `Add-Type`)
- HTA files
- JScript
- VBA / VBS (macros Office)
- Microsoft Office documents

→ Ver [[Weaponization - Payload Formats]].

## Modules — el corazón del framework

Cada C2 tiene su lenguaje propio para modules:

| Framework | Lenguaje |
|---|---|
| Cobalt Strike | Aggressor Script |
| PowerShell Empire | Multi-lenguaje |
| Metasploit | Ruby |

### Post-exploitation modules
Todo post-compromiso inicial: desde `SharpHound.ps1` (enum AD) hasta LSASS dump + credential parsing.

### Pivoting modules
Acceso a segmentos de red restringidos. Típico: **SMB Beacon** — una víctima con admin access actúa como proxy vía SMB para llegar a hosts que no tienen internet directo.

```
[Victim restringido] → SMB → [Victim no-restringido] → HTTPS → [C2 Server]
```

## Facing the world — esconder infraestructura

### Domain Fronting
Abusás CDN (Cloudflare típicamente). El tráfico **parece** ir a Cloudflare (IP, geolocation, cert), pero el `Host` header real apunta al C2.

```
Victim → cloudflare.com (aparente) → [CDN inspecciona Host header] → C2 real
```

> [!warning]
> Muchos CDN (Cloudflare, Google, Azure) **ya bloquearon** domain fronting clásico post-2018. Sigue funcionando en CDNs menos monitoreados, o con variaciones (SNI fronting, fastly, etc.).

### C2 Profiles / Redirectors
Nombres distintos para la misma idea: **NGINX reverse proxy**, **Apache mod_rewrite**, **Malleable HTTP C2 profiles**.

Idea core: el C2 server responde diferente según características del request.

```
Request con header X-C2-Server:  → C2 responde con beacon comms
Request sin ese header:          → C2 responde con página web genérica (ej: clon de Cloudflare)
```

Resultado: si un analista del SOC investiga la IP, ve un sitio legítimo. Solo el agent con el header/firma correcta recibe comandos C2.

> [!warning] HTTPS y extracción de headers
> Si el tráfico es HTTPS end-to-end, el reverse proxy NO puede inspeccionar headers sin terminar TLS. Malleable profiles compensan operando en otra capa (URIs, user-agents, cookies visibles en TLS SNI, etc.).

## Flujo mental — cómo encaja todo

```
1. Operator genera payload (staged/stageless, formato)
2. Payload llega a víctima (phishing, exploit, etc.)
3. Agent beaconea al listener (con sleep+jitter)
4. Tráfico pasa por redirector/CDN (domain fronting + C2 profile)
5. C2 server recibe beacon, operator envía comandos
6. Agent ejecuta, devuelve output
7. Pivoting modules permiten alcanzar redes restringidas
```

## Conexión con planning

- **CONOPS** especifica qué C2 framework se usa a alto nivel
- **Resource Plan** lista infra (C2 servers, redirectors, dominios aged)
- **CTI** define TTPs de C2 del actor emulado (ej: APT29 usa Cobalt Strike con profile X, beacon cada Y min)

→ Si emulás APT28 con Cobalt Strike default profile, rompés fidelidad. El profile y el jitter **deben** replicar al actor.

## Heurísticas del Blue team (lo que te van a cazar)

- Beacons con intervalo regular (sin jitter)
- DNS queries periódicas a dominios low-reputation
- User-agents atípicos o inconsistentes
- Destinos de larga duración sin actividad "humana" (click, scroll)
- TLS fingerprint (JA3/JA3S) del agent vs browser real
- Procesos haciendo network calls inusuales (ej: `notepad.exe` saliendo a internet)

> [!danger]
> Un SOC con SIEM decente + threat hunter capaz detecta Cobalt Strike default en minutos. Por eso existen Malleable profiles + custom tooling.