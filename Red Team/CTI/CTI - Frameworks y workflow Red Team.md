---
tags: [cti, red-team, frameworks, mitre]
created: 2026-04-19
related:
  - "[[CTI - Consumo y perspectivas Blue vs Red]]"
  - "[[Cyber Kill Chain]]"
---

# CTI — Frameworks y workflow Red Team

> [!abstract] TL;DR
> Frameworks dan TTPs categorizados. Workflow: actor → TTPs → kill chain → ejecución en personaje. **CTI = trabajo pre-engagement.**

## Frameworks clave
| Framework | Uso |
|---|---|
| **MITRE ATT&CK** | Matriz global TTPs + grupos. La biblia. |
| **TIBER-EU** | Framework UE para red team intelligence-led en banca |
| **OST Map** | Mapea tools ofensivas a TTPs |

## Categorización de TTPs
Pivoteás por:
1. Threat Group → "dame todo de APT29"
2. Kill Chain Phase → "initial access"
3. Tactic → "credential access"
4. Objective → "exfil de PII"

## Workflow operativo
```
1. Cliente define adversario (vía objectives)
2. CTI team extrae TTPs completos del grupo
3. Mapea al Cyber Kill Chain
4. Traduce en plan táctico + tooling
5. Ejecución: operar "en personaje"
```

## Planning vs Execution — crítico

> [!warning]
> TTPs son **técnica de planificación**, no de ejecución. El operator NO lee Mandiant mientras pivotea.

| Fase | Rol CTI |
|---|---|
| Pre-engagement | Se mastica: se eligen TTPs, arman playbooks, config tooling |
| Durante ejecución | Ya internalizado. Ejecuta. |

Teams grandes → **CTI operator dedicado**. Teams chicos → un operator hace ambos (y sufre).

## Qué modifica CTI en ejecución
- **Tooling** → si APT28 usa X-Agent, no vas con CS vanilla; customizás firmas similares
- **Traffic** → beacon intervals, jitter, protocolos (DNS/HTTPS/domain fronting) replicando al actor
- **Behavior** → horarios (APTs estatales = horario laboral de su zona), targets de lateral, persistencia específica

> [!danger] Fidelidad > creatividad
> Si APT38 usa `schtasks` y metés WMI → rompiste emulación. Blue no audita defensa contra APT38, audita defensa contra *vos*.

## Pentest vs Adversary Emulation
Delivery NO es "rompimos todo". Es "rompimos todo **como lo haría el actor X**, midiendo detección específica contra esa amenaza".