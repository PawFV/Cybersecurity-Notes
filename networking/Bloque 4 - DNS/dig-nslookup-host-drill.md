---
title: dig, nslookup, host y drill
tags: [networking, dns, troubleshooting, dig, nslookup]
aliases: [Herramientas DNS CLI, DNS Lookup Tools, drill]
created: 2026-04-23
difficulty: Básico
related:
  - "[[dns-jerarquia-resolucion]]"
  - "[[dns-records-A-AAAA-NS-MX-TXT-SRV-CNAME]]"
  - "[[dns-over-tls-https-doh-dot]]"
---

# dig, nslookup, host y drill

## TL;DR

- `dig` es la herramienta de referencia para diagnóstico DNS serio en Unix/Linux.
- `nslookup` sigue existiendo, es útil por compatibilidad y en Windows, pero para troubleshooting fino suele quedarse corto.
- `host` es práctico para consultas rápidas y legibles.
- `drill` ofrece sintaxis parecida a `dig` y buenas capacidades para DNSSEC en entornos que usan ldns.
- Si no sabés qué usar: `dig` para precisión, `host` para velocidad, `nslookup` para entornos heredados o Windows.

## Concepto

Estas herramientas no "resuelven distinto" por magia; todas interrogan infraestructura DNS, pero lo hacen con diferentes niveles de detalle, salida y control sobre flags del protocolo.

Lo importante no es memorizar veinte switches, sino entender qué pregunta hace cada comando:

- "¿Qué valor devuelve el resolver que uso?"
- "¿Qué responde este nameserver concreto?"
- "¿Qué pasa si fuerzo TCP?"
- "¿La cadena de delegación cierra?"
- "¿Estoy viendo caché o autoridad?"

Cuando un sysadmin usa mal estas herramientas, suele terminar diagnosticando al resolver equivocado o confundiendo "respuesta del recursivo" con "verdad del autoritativo".

## Cómo funciona

### `dig`

`dig` (Domain Information Groper) expone con bastante fidelidad la estructura real de una respuesta DNS:

- sección de pregunta;
- flags;
- sección de respuesta;
- sección de autoridad;
- sección adicional;
- tiempos, servidor consultado y tamaño.

Ejemplo:

```bash
dig www.example.com A
```

Esto te deja ver no solo el valor, sino el contexto del protocolo.

### `nslookup`

`nslookup` es ampliamente conocido porque viene disponible en muchos sistemas y admins Windows lo usan desde hace décadas. Puede servir para pruebas simples y para consultas interactivas:

```powershell
nslookup
> server 192.168.1.53
> set type=MX
> example.com
```

Funciona, pero para inspección fina de TTL, flags, EDNS, DNSSEC, trace y automatización, `dig` suele ser mejor.

### `host`

`host` apunta a consultas cortas y salida compacta:

```bash
host example.com
host -t MX example.com
host -a example.com
```

Es ideal cuando querés una lectura rápida sin tanta verborragia.

### `drill`

`drill` forma parte del ecosistema `ldns` y es especialmente cómodo para pruebas con foco DNSSEC o para entornos donde ya viene instalado:

```bash
drill example.com A
drill -D example.com
```

No reemplaza universalmente a `dig`, pero es una herramienta perfectamente válida.

### Vista comparativa

```ascii
Herramienta   Mejor uso
-----------   ---------------------------------------------------------
dig           Troubleshooting serio, automatizacion, +trace, +dnssec
nslookup      Compatibilidad, Windows, pruebas simples e interactivas
host          Lookups rapidos y salida compacta
drill         Diagnostico DNS/DNSSEC en entornos basados en ldns
```

## Comandos / configuración

### Consultas basicas con `dig`

```bash
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig -x 203.0.113.25
```

### Salida breve

```bash
dig +short example.com A
dig +short example.com MX
```

### Elegir nameserver explícitamente

```bash
dig @192.168.1.53 www.example.com A
dig @ns1.example.com example.com SOA
```

### Seguir delegación completa

```bash
dig +trace www.example.com
```

### Forzar TCP y pedir DNSSEC

```bash
dig +tcp +dnssec example.com A
```

### `host`

```bash
host example.com
host -t NS example.com
host -a example.com
```

### `drill`

```bash
drill example.com A
drill @192.168.1.53 example.com MX
drill -D example.com
```

### `nslookup` en Windows o entornos mixtos

```powershell
nslookup www.example.com 192.168.1.53
Resolve-DnsName -Name www.example.com -Type A -Server 192.168.1.53
```

> [!tip]
> En Windows moderno, `Resolve-DnsName` suele darte mejor salida que `nslookup`, especialmente para scripting en PowerShell.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Distintas respuestas según el comando | No todos consultan al mismo resolver o alguno usa caché/sufijos de búsqueda del sistema | `dig @192.168.1.53`, `resolvectl status`, `Resolve-DnsName -Server ...` |
| `+short` devuelve vacío pero el dominio existe | Preguntaste el tipo equivocado o no hay answer section para ese tipo | `dig example.com SOA`, `dig example.com MX` |
| El recursivo responde, el autoritativo no | Problema en el servidor autoritativo, firewall o reachability | `dig @ns1.example.com example.com SOA` |
| `SERVFAIL` solo con `+dnssec` | Validación DNSSEC rota o cadena incompleta | `dig +dnssec example.com A`, `drill -D example.com` |
| `nslookup` "anda" pero el problema sigue | La herramienta resolvió algo, pero no necesariamente el FQDN/tipo/servidor correcto | Repetir con `dig` especificando tipo y servidor |

### Patrones útiles de diagnóstico

1. Primero consultá al resolver corporativo.
2. Después pegale directo al autoritativo.
3. Si no cierra, usá `+trace`.
4. Si sospechás truncamiento o middleboxes, repetí con `+tcp`.
5. Si hay sospecha de DNSSEC, sumá `+dnssec` o `drill -D`.

> [!warning]
> No diagnostiques producción con `ping nombre` como única prueba. `ping` te esconde tipo de registro, TTL, flags y servidor efectivo. Sirve como señal débil, no como análisis.

## Seguridad / ofensiva

Estas herramientas también son instrumentos de reconocimiento. Usadas con criterio, permiten inventariar superficie sin generar ruido de aplicación.

### Qué puede obtener un atacante

- `NS`, `MX`, `TXT` y `SOA` para perfilar proveedores y arquitectura;
- reversos (`PTR`) para naming conventions y segmentación;
- intentos de `AXFR` para detectar transferencias de zona mal cerradas;
- TTLs para inferir caché, balanceo o cambios recientes.

### Ejemplos de consultas útiles en assessment autorizado

```bash
dig example.com NS
dig example.com MX
dig example.com TXT
dig -x 203.0.113.25
dig @ns1.example.com example.com AXFR
```

> [!danger]
> Un `AXFR` exitoso contra un autoritativo expone mucho más que nombres públicos lindos: suele revelar hosts administrativos, staging, VPNs, servicios de backup y convenciones de naming internas.

### Lado defensivo

- Registrar intentos de `AXFR` no autorizados.
- Limitar rate de consultas anómalas por origen.
- Monitorear queries de alto volumen sobre subdominios inexistentes.
- Correlacionar enumeración DNS con escaneo posterior en L3/L4.

## Relacionado

- [[dns-jerarquia-resolucion]]
- [[dns-records-A-AAAA-NS-MX-TXT-SRV-CNAME]]
- [[dns-over-tls-https-doh-dot]]
- [[dns-tunneling-iodine-dnscat2]]

## Referencias

- `man 1 dig`
- `man 1 host`
- `man 1 nslookup`
- `man 1 drill`
- BIND 9 Administrator Reference Manual
- ldns official documentation
- Microsoft documentation: `Resolve-DnsName`
