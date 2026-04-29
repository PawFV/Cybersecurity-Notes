---
tags: [red-team/planning, engagement, documentation]
created: 2026-04-19
related:
  - "[[Red Team - CONOPS]]"
---

# Resource Plan

> [!abstract] TL;DR
> Shopping list + calendario fragmentado por fase. Bullets secos. *Quién, cuándo, con qué*.

## Diferencia vs CONOPS
| CONOPS | Resource Plan |
|---|---|
| Prosa semi-técnica | **Bullets puros** |
| Audiencia mixta | Interno/logístico |
| Qué a alto nivel | Fechas, gear, skills |

## Estructura

### Header
- Autor/es, fechas, cliente

### Engagement Dates (por fase)
- Recon
- Initial Compromise
- Post-Exploitation & Persistence
- Misc (reporting, debriefs)

### Knowledge Required *(opcional)*
Skills por fase → detecta **gaps del team** antes de arrancar:
- Recon → OSINT, DNS, cloud enum
- Initial Compromise → phishing infra, payload dev, EDR evasion
- Post-Ex → AD abuse, lateral movement, C2 ops

### Resource Requirements
| Categoría | Ejemplos |
|---|---|
| Personnel | 2 operators, 1 infra lead, 1 OSINT |
| Hardware | Drop boxes, Wi-Fi Pineapple, laptops dedicadas |
| Cloud | Redirectors, C2 servers, dominios aged, VPS |
| Misc | Licencias (CS), VPNs, payloads firmados |

## Regla de oro
"Enough to gather what's required — not overbearing." Si justificás hardware en 3 párrafos, mal.