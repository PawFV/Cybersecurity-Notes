---
title: DNS Records: A, AAAA, NS, MX, TXT, SRV, CNAME
tags: [networking, dns, records, zonefiles, mail]
aliases: [DNS RR Types, Resource Records, Tipos de Registros DNS]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[dns-jerarquia-resolucion]]"
  - "[[dig-nslookup-host-drill]]"
  - "[[dns-caching-forwarders-dnsmasq-unbound]]"
---

# DNS Records: A, AAAA, NS, MX, TXT, SRV, CNAME

## TL;DR

- Un registro DNS no es "una IP"; es una **tupla tipada** dentro de una zona.
- `A` y `AAAA` resuelven hosts a IPv4/IPv6.
- `NS` delega autoridad; `MX` define correo; `TXT` transporta metadata; `SRV` publica ubicación de servicios; `CNAME` crea alias.
- El error clásico es mezclar semánticas: usar `CNAME` donde hace falta un registro directo, o publicar `MX` apuntando a un alias mal resuelto.
- En seguridad, estos registros exponen inventario, proveedores, dependencias externas y oportunidades de takeover.

## Concepto

En DNS, cada registro es un **Resource Record (RR)** asociado a un nombre, un tipo, una clase (casi siempre `IN`) y un TTL. Pensarlo como "la entrada DNS de un host" se queda corto: un mismo nombre puede tener varios tipos distintos.

Ejemplo simplificado:

```text
www.example.com.   300  IN A      203.0.113.20
www.example.com.   300  IN AAAA   2001:db8::20
example.com.      3600  IN MX 10  mail.example.com.
_sip._tcp.example.com. 300 IN SRV 10 60 5060 sip01.example.com.
```

Cada tipo responde una pregunta diferente:

- `A`: "¿Cuál es la IPv4 de este nombre?"
- `AAAA`: "¿Cuál es la IPv6?"
- `NS`: "¿Quién manda sobre esta zona?"
- `MX`: "¿A qué servidor le entrego correo?"
- `TXT`: "¿Qué metadata textual publica este nombre?"
- `SRV`: "¿En qué host y puerto vive este servicio?"
- `CNAME`: "Este nombre en realidad es alias de otro".

## Cómo funciona

### 1. Registro `A`

Mapea un nombre a una dirección IPv4.

```text
app.example.com. 300 IN A 203.0.113.10
```

Uso típico:

- frontends web;
- VIPs de load balancers;
- nombres de administración;
- servicios que todavía operan solo con IPv4.

### 2. Registro `AAAA`

Mapea un nombre a una dirección IPv6.

```text
app.example.com. 300 IN AAAA 2001:db8:10::10
```

Si publicás `AAAA`, asumí que hay camino IPv6 funcional, no solo dirección configurada.

> [!warning]
> Publicar `AAAA` "porque queda lindo" cuando el servicio no funciona bien por IPv6 genera fallas intermitentes difíciles de reproducir. Muchos clientes intentarán IPv6 primero.

### 3. Registro `NS`

Define qué servidores son autoritativos para una zona o una delegación.

```text
example.com.      86400 IN NS ns1.example.com.
example.com.      86400 IN NS ns2.example.com.
dev.example.com.  86400 IN NS ns1.dev-dns.example.net.
```

Cuando aparece en una delegación, `NS` no "resuelve un host": indica a quién consultar para seguir la cadena de autoridad.

### 4. Registro `MX`

Define a qué hosts se entrega correo para un dominio, con preferencia numérica menor = prioridad mayor.

```text
example.com.      3600 IN MX 10 mail1.example.com.
example.com.      3600 IN MX 20 mail2.example.com.
```

Detalles importantes:

- el destino de un `MX` debe ser un nombre que luego resuelva a `A` y/o `AAAA`;
- conviene evitar cadenas raras de alias;
- `MX` vive en el dominio receptor, no en el host del servidor.

### 5. Registro `TXT`

Transporta texto arbitrario. Hoy se usa sobre todo para políticas y validaciones.

```text
example.com. 300 IN TXT "v=spf1 ip4:203.0.113.0/24 -all"
_dmarc.example.com. 300 IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
selector1._domainkey.example.com. 300 IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkq..."
```

Casos comunes:

- SPF;
- DKIM;
- DMARC;
- verificaciones de propiedad para SaaS/CDN/CA.

### 6. Registro `SRV`

Publica ubicación de servicio, con prioridad, peso y puerto.

```text
_ldap._tcp.example.com. 300 IN SRV 0 100 389 dc1.example.com.
_kerberos._tcp.example.com. 300 IN SRV 0 100 88 kdc1.example.com.
```

Formato:

```ascii
_servicio._protocolo.nombre  TTL  IN  SRV  prioridad peso puerto destino
```

Muchos clientes empresariales, especialmente en entornos Microsoft/LDAP/SIP/XMPP, dependen de esto para autodiscovery.

### 7. Registro `CNAME`

Declara que un nombre es alias canónico de otro.

```text
www.example.com. 300 IN CNAME edge.example.net.
```

Regla operacional clave: un nombre que tiene `CNAME` no puede tener otros registros de datos coexistiendo en el mismo owner name.

No hagas esto:

```text
www.example.com. 300 IN CNAME edge.example.net.
www.example.com. 300 IN A 203.0.113.10
```

Eso es inválido a nivel de diseño de zona.

> [!tip]
> `CNAME` es cómodo para abstraer backends cambiantes, pero en apex de zona (`example.com.`) muchas plataformas y DNS servers imponen restricciones. Ahí suelen aparecer soluciones tipo ALIAS/ANAME, que no son tipos estándar del protocolo.

## Comandos / configuración

### Consultas por tipo

```bash
dig example.com A
dig example.com AAAA
dig example.com NS
dig example.com MX
dig example.com TXT
dig _ldap._tcp.example.com SRV
dig www.example.com CNAME
```

### Respuestas compactas

```bash
dig +short example.com MX
dig +short example.com TXT
dig +short _sip._tcp.example.com SRV
```

### Consultar todos los datos útiles de un nombre

```bash
host -a example.com
```

### Ejemplo de fragmento de zona

```dns
$ORIGIN example.com.
$TTL 300

@       IN SOA ns1.example.com. dns-admin.example.com. (
            2026042301 ; serial
            3600       ; refresh
            900        ; retry
            1209600    ; expire
            300        ; negative cache TTL
)

@       IN NS    ns1.example.com.
@       IN NS    ns2.example.com.
@       IN MX 10 mail.example.com.

www     IN A     203.0.113.20
www     IN AAAA  2001:db8:20::20
mail    IN A     203.0.113.25
_sip._tcp IN SRV 10 50 5060 sip01.example.com.
@       IN TXT   "v=spf1 ip4:203.0.113.0/24 -all"
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| El sitio abre por IPv4 pero no por IPv6 | `AAAA` publicado hacia un servicio roto o sin ruta IPv6 válida | `dig app.example.com AAAA` y prueba con `curl -6` |
| El correo rebota o entra en cola | `MX` apunta a un host sin `A`/`AAAA`, con firewall o nombre mal resuelto | `dig example.com MX` y luego `dig mail.example.com A AAAA` |
| Un cliente no descubre el servicio automáticamente | `SRV` con puerto, prioridad o nombre incorrectos | `dig _ldap._tcp.example.com SRV` |
| Un cambio de CDN/SaaS no impacta | `CNAME` correcto pero caché vigente o alias encadenado a un destino viejo | `dig www.example.com CNAME +ttlunits` |
| Validaciones de correo fallan | `TXT` SPF/DKIM/DMARC malformado o truncado | `dig example.com TXT`, `dig selector1._domainkey.example.com TXT` |

### Errores de modelado frecuentes

1. Publicar `MX` apuntando a un nombre que no existe.
2. Poner `CNAME` y otros registros en el mismo nombre.
3. Crear `SRV` sin respetar formato `_servicio._protocolo`.
4. Olvidar que `NS` de delegación implica también reachability real hacia esos servidores.

## Seguridad / ofensiva

Los registros DNS son una mina de información para reconocimiento y un punto de control crítico para defensa.

### Lo que expone cada tipo

- `A` / `AAAA`: superficie expuesta, balanceo, hosting dual-stack, rangos usados.
- `NS`: proveedor DNS, secundarias, terceros delegados.
- `MX`: proveedor de correo, gateways, prioridades de contingencia.
- `TXT`: políticas de correo, tenants SaaS, validaciones de dominio.
- `SRV`: servicios internos o híbridos que a veces revelan AD, SIP, XMPP o VoIP.
- `CNAME`: dependencias externas y posibles recursos huérfanos.

### Riesgos típicos

- **Dangling CNAME** hacia recursos borrados en cloud/CDN/SaaS.
- **SPF demasiado permisivo** (`+all`, demasiados includes, delegaciones difusas).
- **TXT con secretos por error**: tokens, challenge strings, metadata sensible.
- **SRV expuestos externamente** revelando servicios que el negocio ni sabe que publicó.

> [!danger]
> Un `CNAME` que apunta a un recurso inexistente pero re-registrable por un tercero puede terminar en subdomain takeover. No es un problema teórico: suele aparecer en migraciones mal cerradas.

### Controles recomendados

- Inventariar `TXT`, `MX`, `NS` y `CNAME` en cada cambio.
- Validar que destinos de `MX`, `SRV` y `CNAME` existan y sigan bajo control.
- Revisar exposición externa de registros de autodiscovery.
- Monitorear expiración o baja de recursos cloud referenciados desde DNS.

## Relacionado

- [[dns-jerarquia-resolucion]]
- [[dig-nslookup-host-drill]]
- [[dns-caching-forwarders-dnsmasq-unbound]]

## Referencias

- RFC 1034 - *Domain Names - Concepts and Facilities*
- RFC 1035 - *Domain Names - Implementation and Specification*
- RFC 2181 - *Clarifications to the DNS Specification*
- RFC 2782 - *A DNS RR for Specifying the Location of Services (SRV)*
- RFC 7505 - *A "Null MX" No Service Resource Record for Domains That Accept No Mail*
- RFC 7208 - *Sender Policy Framework (SPF)*
- RFC 7489 - *DMARC*
- RFC 6376 - *DomainKeys Identified Mail (DKIM) Signatures*
- `man 1 dig`
- `man 1 host`
