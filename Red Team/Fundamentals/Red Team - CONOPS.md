---
tags: [red-team/planning, engagement, documentation]
created: 2026-04-19
related:
  - "[[Red Team - Client Objectives y Scope]]"
  - "[[Red Team - Resource Plan]]"
---

# CONOPS (Concept of Operations)

> [!abstract] TL;DR
> Executive summary **pre-ejecución**. Bisagra entre C-level y red cell.

## Audiencia dual
- **Cliente/C-level** → cero-mínimo background técnico
- **Red cell** → lo usan como baseline para planes detallados

→ Escritura **semi-técnica**: alto nivel sin castrar detalles clave.

## Componentes críticos
| Campo                       | Contenido                                      |
| --------------------------- | ---------------------------------------------- |
| Client Name                 | Quién paga                                     |
| Service Provider            | Quién ejecuta                                  |
| Timeframe                   | Ventana temporal                               |
| General Objectives/Phases   | Recon → initial access → lateral → exfil       |
| Other Training Objectives   | Ej: medir SOC response time                    |
| High-Level Tools/Techniques | Cobalt Strike, phishing, C2 frameworks (macro) |
| Threat Group                | APT28, FIN7, Lazarus (si aplica)               |

## Regla de oro
**Just enough information.**
- Demasiado detalle → filtrás TTPs en doc de circulación amplia (mal OpSec)
- Muy poco → red cell sin base para planificar

> [!tip]
> Debe ser **escaneable**: bullets, secciones claras, cero prosa densa. El CISO lo lee en el subte.