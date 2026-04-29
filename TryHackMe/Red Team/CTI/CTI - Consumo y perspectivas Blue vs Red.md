---
tags: [cti, red-team, blue-team, fundamentals]
created: 2026-04-19
related:
  - "[[CTI - Frameworks y workflow Red Team]]"
  - "[[MITRE ATT&CK]]"
---

# CTI — Consumo y perspectivas

> [!abstract] TL;DR
> CTI = IOCs (qué) + TTPs (cómo). Blue lo usa para detectar; Red lo usa para **auditar si el Blue lo usa bien**.

## Consumo de CTI
| Input | Qué aporta |
|---|---|
| **IOCs** | Datos atómicos: dominios, IPs, hashes, strings |
| **TTPs** | Comportamiento: cómo opera el actor |

**Distribución:** ISACs sectoriales (FS-ISAC, H-ISAC) + plataformas (MISP, OpenCTI, ThreatConnect).

> [!note]
> "ISAC" se usa loosely. A veces = plataforma genérica.

## IOC vs TTP — Pyramid of Pain
```
IOC  = "usó guantes marca X"      ← caro cambiar: bajo
TTP  = "entra por el techo..."    ← caro cambiar: alto
```
IOCs caducan rápido. TTPs son **pegajosos** → pegar arriba duele más al adversario.

## Dos perspectivas

### Blue team
- Contextualiza threat landscape
- Construye detecciones (Sigma, YARA, SIEM)
- Cuantifica hallazgos post-incidente

### Red team (el giro)
> CTI desde el rojo = **análisis de qué tan bien el Blue sabe usar SU PROPIO CTI para detectarte.**

Si emulás APT29 fielmente y el SOC no te pesca → problema no es el feed, es **operacionalización**. Ese gap = valor del red team maduro.

## Por qué APTs
- TTPs bien documentados (MITRE ATT&CK Groups, Mandiant, CrowdStrike)
- Operaciones coherentes → emulables
- Habilitan adversary emulation focalizada