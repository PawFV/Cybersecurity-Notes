---
tags: [red-team/planning, engagement, fundamentals]
created: 2026-04-19
source: TryHackMe - Red Team Engagement
related:
  - "[[Red Team - CONOPS]]"
  - "[[Red Team - Resource Plan]]"
---

# Client Objectives y Scope

> [!abstract] TL;DR
> **Objetivos** = qué querés lograr (negociable entre cliente y red team).
> **Scope** = dónde podés pisar (lo define el cliente, punto).

## Objetivos del cliente
Pilar de toda engagement. Sin objetivos claros → campaña desestructurada.
- Se **negocian**, entendimiento mutuo obligatorio
- De ahí salen todos los demás documentos
- Definen **tono** y **enfoque**

### Tipos de engagement
| Tipo | Cuándo |
|---|---|
| Pentest general | TTPs estándar, scope amplio |
| Adversary Emulation | Emulás un APT concreto según industria (banco → APT38) |

## Scope
- Lo define **solo el cliente** (red team puede objetar si rompe la op)
- Define qué NO podés tocar, a veces qué sí

### Ejemplos de verbiage
- ❌ No exfiltrar PII / data
- ❌ Production servers off-limits  
- ❌ Downtime prohibido
- ✅/❌ Rangos IP in/out of scope

## Mindset al leer scope
No literal — leer entre líneas:
- "No downtime" = nada de DoS, cuidado con fuzzing agresivo
- ¿Qué ataques **quedan habilitados** dentro del scope?
- Tenés que poder armar plan solo con objectives + scope

## Jerarquía
`Objectives + Scope` → [[Red Team - CONOPS]] → [[Red Team - Resource Plan]] → Ops Plan