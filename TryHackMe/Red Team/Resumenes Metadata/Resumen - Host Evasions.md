---
tags: [red-team, resumen, metadata, host-evasion]
created: 2026-04-23
source: C:\Users\paujo\Documents\mock\data.json
related:
  - "[[CTI - Frameworks y workflow Red Team]]"
  - "[[MITRE ATT&CK]]"
---

# Resumen - Host Evasions

## Estado

- **Rooms:** `0/11` completadas
- **Tasks:** `1/99` (1.0%)
- **Questions:** `1/27`
- **Avance real:** solo `Windows Internals` iniciado.

## Detalle por room

| Room | Code | Tasks | Questions | Progreso |
|---|---|---:|---:|---:|
| Windows Internals | `windowsinternals` | 1/8 | 1/27 | 3% |
| Introduction to Windows API | `windowsapi` | 0/10 | 0/0 | 0% |
| Abusing Windows Internals | `abusingwindowsinternals` | 0/8 | 0/0 | 0% |
| Introduction to Antivirus | `introtoav` | 0/8 | 0/0 | 0% |
| AV Evasion: Shellcode | `avevasionshellcode` | 0/11 | 0/0 | 0% |
| Obfuscation Principles | `obfuscationprinciples` | 0/9 | 0/0 | 0% |
| Signature Evasion | `signatureevasion` | 0/8 | 0/0 | 0% |
| Bypassing UAC | `bypassinguac` | 0/8 | 0/0 | 0% |
| Runtime Detection Evasion | `runtimedetectionevasion` | 0/9 | 0/0 | 0% |
| Evading Logging and Monitoring | `monitoringevasion` | 0/11 | 0/0 | 0% |
| Living Off the Land | `livingofftheland` | 0/9 | 0/0 | 0% |

## Enfoque recomendado

- Priorizar este orden:
  1. `Windows API`
  2. `Antivirus + Signature Evasion`
  3. `Runtime/Logging Evasion`
  4. `Living Off the Land`
- Objetivo: llegar a `>=40%` del modulo antes de abrir nuevos tracks.

> [!warning]
> Este modulo impacta directo en OPSEC y evasión realista. Dejarlo muy atrás limita la calidad de simulación en ejercicios AD y C2.

