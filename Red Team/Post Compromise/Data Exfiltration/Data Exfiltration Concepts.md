---
tags: [red-team, data-exfiltration, c2, tunneling, concepts]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Exfiltration using TCP Socket]]"
  - "[[Exfiltration using SSH]]"
  - "[[Exfiltration using HTTP(S)]]"
  - "[[Exfiltration using ICMP]]"
  - "[[Exfiltration over DNS]]"
  - "[[DNS Tunneling]]"
---

# Data Exfiltration — Conceptos

> [!abstract] TL;DR
> Exfiltración = sacar datos sensibles de la red comprometida hacia infraestructura del atacante **disimulando el tráfico dentro de protocolos legítimos** (HTTP/S, DNS, ICMP, SSH, TCP). Ocurre en la fase final del Cyber Kill Chain (*Actions on Objectives*). Tres patrones principales: exfil tradicional (unidireccional), C2 (bidireccional limitado) y tunneling (bidireccional continuo).

## Qué es

**Data Exfiltration** es la técnica post-explotación que consiste en copiar y transferir datos sensibles desde la red comprometida hacia una máquina controlada por el atacante. La clave no es sólo *sacar* los datos sino **hacerlo sin disparar alertas**: el tráfico viaja camuflado como si fuera actividad normal de la red.

Se apoya en protocolos comunes (TCP, HTTP/S, DNS, ICMP, SSH) porque cumplen dos propiedades útiles para el atacante:

1. Están **permitidos por default** en casi todos los firewalls corporativos.
2. No fueron diseñados para transportar archivos, así que los IDS/IPS clásicos suelen **no inspeccionar sus payloads** con la misma profundidad que el tráfico web.

La idea es parásita: aprovechar un canal construido para otra cosa para mover bytes que no deberían salir.

## Posicionamiento en el Kill Chain

```
Recon → Weaponization → Delivery → Exploitation → Installation → C2 → [Actions on Objectives]
                                                                            │
                                                                            └── Data Exfiltration
```

La exfiltración es **el último paso**: el atacante ya tiene acceso persistente, ya escaló privilegios, ya hizo lateral movement y ahora quiere convertir ese acceso en valor (datos que puede vender, filtrar o usar para ransomware).

## Tipos de datos objetivo

| Categoría | Ejemplos | Valor |
|---|---|---|
| Credenciales | usuarios/passwords/tokens API, SSH keys | Acceso a otros sistemas |
| Información financiera | tarjetas, cuentas, transferencias | Monetización directa |
| PII de empleados/clientes | DNI, dirección, salud | Extorsión, phishing dirigido |
| Propiedad intelectual | código fuente, planos, fórmulas | Espionaje industrial |
| Claves criptográficas | certificados, claves privadas | Impersonation, MITM |
| Decisiones de negocio | M&A, resultados no publicados | Insider trading |

## Los tres escenarios de exfiltración

### 1. Exfiltración tradicional (unidireccional)

El atacante sólo quiere **sacar los datos**; no le interesa la respuesta del servidor víctima.

```
[Víctima] ──────── datos ────────▶ [Atacante]
          (una sola dirección)
```

Ejemplo típico: subir un dump de la base a un bucket S3, o enviar un archivo por HTTP POST a un servidor controlado.

### 2. Comunicaciones C2 (bidireccional limitado)

Tráfico en **ambas direcciones pero acotado**: el atacante manda comandos cortos, la víctima responde con resultados. No es un canal de datos continuo.

```
[Víctima] ◀── comando ── [Atacante]
         ── resultado ──▶
         (pulsos cortos, bajo volumen)
```

La víctima hace *beaconing*: cada X segundos consulta si hay comandos nuevos. Este patrón es el corazón de frameworks como Cobalt Strike, Sliver o Metasploit.

### 3. Tunneling (bidireccional continuo)

Se establece un **túnel persistente** que se comporta como una VPN improvisada. Cualquier tráfico (SSH, HTTP, RDP) se encapsula dentro del protocolo de transporte (típicamente DNS o HTTP) y sale al mundo.

```
[Víctima]  ◀══════════════════════▶  [Atacante]
          (tráfico constante bidireccional,
           múltiples protocolos encapsulados)
```

Es la opción más potente pero también la más ruidosa: mover gigabytes dentro de queries DNS genera anomalías de volumen obvias.

## Trade-offs al elegir canal

Ningún canal es gratuito. Siempre se negocia entre tres ejes:

```
        Sigilo
         /\
        /  \
       /    \
      /______\
  Velocidad  Fiabilidad
```

- **Sigilo alto + velocidad baja** → DNS (queries chicas, muchas, detectables por volumen).
- **Velocidad alta + sigilo bajo** → TCP directo (rápido pero protocolo no-estándar = alerta inmediata).
- **Fiabilidad alta + sigilo medio** → HTTPS (funciona siempre, pero deja rastro en proxies).

## Técnicas cubiertas

- [[Exfiltration using TCP Socket]] — socket crudo + `nc` + base64 + EBCDIC.
- [[Exfiltration using SSH]] — `tar` por stdin sobre un canal SSH cifrado.
- [[Exfiltration using HTTP(S)]] — POST a handler PHP + tunneling con Neo-reGeorg.
- [[Exfiltration using ICMP]] — abuso del campo Data de ICMP Echo.
- [[Exfiltration over DNS]] — encoding como subdominios + TXT records para C2.
- [[DNS Tunneling]] — `iodine` encapsulando IP sobre queries DNS.

## Señales que usa el Blue Team

> [!warning]
> Ningún canal es invisible. Los NIDS/EDR modernos detectan exfiltración por:
> - **Volumen anómalo** en protocolos que normalmente son chicos (DNS, ICMP).
> - **Entropía alta** en subdominios o parámetros (indica base64/cifrado).
> - **Beaconing**: patrones temporales regulares (cada 60s ± jitter).
> - **Destinos raros**: dominios jóvenes, ASN poco habitual, geolocalización inesperada.
> - **Inversión de ratios**: un host que normalmente recibe mucho y envía poco empieza a hacer lo contrario.

## Preguntas y Respuestas

- **¿En qué escenario el tráfico continúa enviándose y recibiéndose durante toda la conexión?**
  → `Tunneling`

- **¿En qué escenario el tráfico va en una sola dirección?**
  → `Traditional Data Exfiltration`
