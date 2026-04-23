---
title: DNS: Jerarquía y Resolución
tags: [networking, dns, resolucion, recursion, authoritative]
aliases: [DNS Resolution, Jerarquía DNS, Recursive vs Authoritative]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[dns-records-A-AAAA-NS-MX-TXT-SRV-CNAME]]"
  - "[[dig-nslookup-host-drill]]"
  - "[[dns-caching-forwarders-dnsmasq-unbound]]"
  - "[[dns-over-tls-https-doh-dot]]"
---

# DNS: Jerarquía y Resolución

## TL;DR

- DNS es una base de datos distribuida y jerárquica, no una tabla plana global.
- Los servidores **recursivos** resuelven en nombre del cliente; los **autoritativos** responden por una zona concreta.
- La resolución normal sigue esta cadena: `stub resolver` local -> resolver recursivo -> root -> TLD -> autoritativo.
- El dato importante no es solo la respuesta, sino también su **TTL**, la delegación (`NS`) y el contexto de caché.
- En troubleshooting, distinguir `NXDOMAIN`, `SERVFAIL`, timeout y respuesta vacía evita perder horas mirando el lugar equivocado.

## Concepto

DNS (Domain Name System) traduce nombres legibles por humanos a datos que consumen máquinas: direcciones IP, servidores de correo, aliases, claves de validación y metadata operativa. La idea central es simple: no querés hardcodear `203.0.113.10` en cada cliente, porque las IP cambian y los servicios evolucionan.

La parte importante, y la que suele romperse en producción, es que DNS no funciona como "un servidor mágico". Funciona como una **jerarquía delegada**:

- La **raíz** sabe qué servidores manejan cada TLD.
- El **TLD** sabe qué servidores son autoritativos para un dominio.
- El **autoritativo** sabe los registros concretos de su zona.

Eso permite escalar internet sin una base de datos central monolítica.

> [!note]
> Cuando alguien dice "el DNS está caído", casi nunca significa que "DNS entero" falló. En general falló una de estas piezas: el resolver recursivo, una delegación, el autoritativo, la caché local o la conectividad hacia UDP/TCP 53.

## Cómo funciona

### Actores del proceso

- **Stub resolver:** librería o servicio local que recibe la consulta de la aplicación. En Linux moderno muchas veces interactúa con `systemd-resolved`; en otros entornos habla directo con los nameservers configurados en `/etc/resolv.conf`.
- **Resolver recursivo:** hace el trabajo pesado. Pregunta a otros servidores, sigue delegaciones, cachea respuestas y devuelve el resultado final.
- **Servidor root:** no conoce el registro final, pero sabe quién maneja el TLD.
- **Servidor TLD:** delega hacia los autoritativos del dominio.
- **Servidor autoritativo:** responde con la verdad administrativa de la zona.

### Flujo de resolución recursiva

```ascii
Aplicacion
    |
    v
Stub Resolver local
    |
    v
Resolver Recursivo
    |
    +--> Root (.) ---------> "Para .com preguntale a estos NS"
    |
    +--> TLD (.com) -------> "Para example.com preguntale a estos NS"
    |
    +--> Autoritativo -----> "www.example.com = 203.0.113.20"
    |
    v
Respuesta al cliente + TTL
```

### Recursión vs iteración

Desde el punto de vista del cliente, normalmente la consulta al resolver recursivo es **recursiva**: "resolvemelo vos". En cambio, entre servidores DNS el diálogo suele ser **iterativo**: cada servidor responde con la mejor pista que tiene.

Ejemplo conceptual:

1. El cliente pregunta por `www.example.com`.
2. El resolver no lo tiene en caché.
3. Pregunta a un root por `www.example.com`.
4. El root responde con referencias a `.com`.
5. El resolver pregunta al TLD `.com`.
6. El TLD responde con los `NS` de `example.com`.
7. El resolver pregunta al autoritativo de `example.com`.
8. El autoritativo responde el registro `A` o `AAAA`.
9. El resolver cachea y devuelve la respuesta.

### Delegación y zona

Una **zona** es la porción del namespace que administra un conjunto de servidores autoritativos. Un dominio puede delegar subdominios a otras zonas:

```ascii
. 
└── com
    └── example.com
        ├── www.example.com
        ├── mail.example.com
        └── dev.example.com   -> delegada a otros NS
```

Eso implica algo importante: el hecho de que `example.com` exista no garantiza que `dev.example.com` esté servido por los mismos servidores ni con las mismas políticas.

### TTL y caché

Cada respuesta viene con un **Time To Live**. No es decoración: le dice al resolver cuánto tiempo puede reutilizar esa respuesta sin volver a preguntarle al autoritativo.

- TTL bajo: más frescura, más carga sobre autoritativos.
- TTL alto: menos carga, cambios más lentos en propagarse.

> [!tip]
> Antes de una migración de IP o MX, bajar el TTL con anticipación suele ser más importante que tocar el registro el mismo día. Si no lo hiciste antes, ya llegaste tarde para esa ventana.

## Comandos / configuración

### Ver qué resolver usa el sistema

```bash
resolvectl status

cat /etc/resolv.conf
```

### Resolver un nombre y ver la respuesta final

```bash
dig example.com A

dig www.example.com AAAA
```

### Seguir la cadena completa de delegación

```bash
dig +trace www.example.com
```

Salida esperable, simplificada:

```text
.              518400  IN NS a.root-servers.net.
com.           172800  IN NS a.gtld-servers.net.
example.com.    86400  IN NS ns1.example.com.
www.example.com. 300   IN A 203.0.113.20
```

### Consultar directo a un autoritativo

```bash
dig @ns1.example.com www.example.com A
```

### Forzar consulta TCP

```bash
dig +tcp example.com MX
```

Esto sirve cuando:

- la respuesta excede el tamaño cómodo de UDP;
- hay truncamiento (`TC=1`);
- sospechás filtrado o middleboxes raros sobre UDP 53.

### Windows

```powershell
Resolve-DnsName -Name www.example.com -Type A

nslookup www.example.com
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| `NXDOMAIN` | El nombre consultado no existe en esa zona. No es un problema de reachability sino de dato. | `dig www.example.com A` |
| `SERVFAIL` | El resolver no pudo completar la resolución. Suele ser DNSSEC roto, delegación inconsistente o autoritativo inaccesible. | `dig +dnssec www.example.com A` y `dig +trace www.example.com` |
| Timeout | No hay respuesta desde el resolver o algún salto está filtrando UDP/TCP 53. | `dig @192.168.1.53 example.com A` y captura con `tcpdump -ni eth0 port 53` |
| Respuesta vieja después de un cambio | Caché local, del recursivo o TTL alto todavía vigente. | `dig example.com A +ttlunits` |
| `NOERROR` pero sin answers | La zona existe, pero no hay ese tipo de registro o estás frente a una delegación/corte distinto. | `dig example.com MX`, `dig example.com SOA` |

### Checklist práctico

1. Confirmá si falla desde una app o desde el resolver.
2. Verificá cuál es el nameserver efectivo del host.
3. Probá contra el resolver corporativo y contra el autoritativo por separado.
4. Hacé `+trace` para detectar en qué nivel se corta la cadena.
5. Si hay DNSSEC, revisá validación y reloj del sistema.

> [!warning]
> Un `ping` fallando por nombre no siempre prueba un problema DNS. Podés tener resolución correcta y falla posterior en routing, firewall o servicio. Separá resolución de conectividad.

## Seguridad / ofensiva

Desde Red Team y DevSecOps, DNS importa por tres motivos: es ubicuo, atraviesa casi todos los perímetros y produce muchísimo metadata útil.

### Superficie de ataque habitual

- **Transferencias de zona abiertas (`AXFR`)**: exposición de inventario interno o externo.
- **Open resolvers**: abuso para amplificación/reflexión DDoS.
- **Cache poisoning**: insertar respuestas falsas en la caché del recursivo.
- **Dangling delegations**: `NS` o `CNAME` apuntando a recursos huérfanos reutilizables por un atacante.
- **Split-horizon mal diseñado**: filtrado accidental de nombres internos hacia vistas externas.

### Qué mira un atacante

- nombres de hosts (`vpn`, `git`, `jira`, `mail`, `sip`);
- patrones de segmentación (`prod`, `stage`, `corp`, `dmz`);
- servidores de correo y proveedores externos;
- TTLs y cambios frecuentes que delatan infraestructura dinámica o failover.

> [!danger]
> Permitir recursión a internet en un resolver no autenticado convierte al servicio en infraestructura utilizable por terceros. Eso es un problema de abuso, reputación y disponibilidad, no solo de "mala práctica".

### Controles defensivos básicos

- Restringir recursión por ACL.
- Deshabilitar `AXFR` salvo entre secundarios autorizados.
- Firmar zonas sensibles con DNSSEC cuando el modelo operativo lo soporte.
- Monitorear cambios de `NS`, `MX`, `TXT` y `CNAME`.
- Registrar queries anómalas por longitud, entropía y volumen.

## Relacionado

- [[dns-records-A-AAAA-NS-MX-TXT-SRV-CNAME]]
- [[dig-nslookup-host-drill]]
- [[dns-caching-forwarders-dnsmasq-unbound]]
- [[dns-over-tls-https-doh-dot]]
- [[dns-tunneling-iodine-dnscat2]]

## Referencias

- RFC 1034 - *Domain Names - Concepts and Facilities*
- RFC 1035 - *Domain Names - Implementation and Specification*
- RFC 2308 - *Negative Caching of DNS Queries (DNS NCACHE)*
- RFC 4033 - *DNS Security Introduction and Requirements*
- `man 1 dig`
- systemd documentation: `resolvectl(1)` and `systemd-resolved.service(8)`
