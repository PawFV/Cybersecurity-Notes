---
tags: [red-team, c2, frameworks, tools]
created: 2026-04-19
source: TryHackMe - Intro to C2
related:
  - "[[C2 Framework - Fundamentos]]"
  - "[[CTI - Frameworks y workflow Red Team]]"
---

# C2 Frameworks comunes

> [!abstract] TL;DR
> Free = bien documentados, signatures públicas → más detectables. Paid = evasión superior, features avanzadas, updates rápidos. Elegir según objetivo, presupuesto y fidelidad al actor emulado.

## Free vs Paid — criterio

| Free | Paid |
|---|---|
| Signatures conocidas por AV/EDR | Evasión más fresca |
| Community-driven | Features on-demand del vendor |
| Bueno para aprender / labs | Bueno para engagements reales |
| Ej: Metasploit detectado por cualquier EDR decente | Ej: CS ofrece **VPN tunnel desde beacon** (único) |

## Free

### Metasploit (Rapid7)
- El más conocido. Incluido en Kali.
- Exploitation + post-ex. MSFVenom como payload generator.
- **Detectabilidad alta** — firmas públicas por todos lados.

### Armitage
- GUI en Java para Metasploit.
- Misma familia que Cobalt Strike (mismo autor original: Raphael Mudge).
- Feature distintivo: **Hail Mary** — tira todos los exploits aplicables contra un host.


### PowerShell Empire / Starkiller
- Originalmente Veris Group, ahora mantenido por BC Security.
- Agents multi-lenguaje, multi-plataforma.
- Starkiller = GUI de Empire.

### Covenant (Ryan Cobb)
- Escrito en **C#** → raro en el ecosistema.
- Foco: post-exploitation y lateral movement.
- Listeners HTTP/HTTPS/SMB.

### Sliver (Bishop Fox)
- Escrito en **Go** → implants difíciles de reversear.
- Multi-user, CLI-based.
- Protocolos: WireGuard, mTLS, HTTP(S), DNS.
- Features: BOF support, DNS Canary domains, auto Let's Encrypt.

> [!tip]
> Sliver es el "serious free option" para engagements reales. El más competitivo contra las opciones paid.

## Paid

### Cobalt Strike (Help Systems, originalmente Mudge)
- El estándar de facto en red team pro.
- Java, altamente flexible.
- **Aggressor Script** para extender.
- También el más emulado por APTs (APT29, FIN7, Lazarus) → cracked versions circulan en el submundo criminal.

### Brute Ratel (Chetan Nayak)
- "C4" — Customizable Command and Control Center.
- Marketing: adversary simulation real.
- Alternativa moderna a CS con mejor evasión out-of-the-box.

## Recurso clave

**C2 Matrix** (Jorge Orchilles + Bryson Bort) → comparativa exhaustiva de todos los C2 por features, protocolos, OPSEC, etc. Consulta obligada al elegir framework para un engagement.

## Criterio de elección — red teamer pragmático

1. **¿Qué actor emulás?** → framework que use ese actor (APT29 → CS; TA505 → custom; etc.)
2. **¿Qué EDR tiene el target?** → CS cracked vs CrowdStrike = te caza rápido
3. **¿Budget?** → Sliver cubre 80% de casos sin gastar
4. **¿Team familiarizado?** → meter BRC4 sin training previo = mal día