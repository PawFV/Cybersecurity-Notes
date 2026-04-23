---
title: DNS Caching, Forwarders, dnsmasq y unbound
tags: [networking, dns, cache, forwarder, dnsmasq]
aliases: [DNS Forwarders, dnsmasq, unbound, Cache DNS]
created: 2026-04-23
difficulty: Avanzado
related:
  - "[[dns-jerarquia-resolucion]]"
  - "[[dig-nslookup-host-drill]]"
  - "[[dns-over-tls-https-doh-dot]]"
---

# DNS Caching, Forwarders, dnsmasq y unbound

## TL;DR

- Un cache DNS reduce latencia, tráfico repetido y dependencia de upstreams.
- Un **forwarder** reenvía consultas a otro resolver; un **resolver recursivo** resuelve por sí mismo.
- `dnsmasq` es liviano, práctico y excelente para labs, edge, hosts y redes chicas.
- `unbound` es más robusto para recursión, validación DNSSEC, políticas y despliegues serios.
- El error clásico es apilar componentes (`NetworkManager`, `systemd-resolved`, `dnsmasq`, `unbound`) sin definir quién manda realmente.

## Concepto

Cuando un cliente consulta siempre a resolvers externos, cada lookup implica latencia, ruido y dependencia. Un cache local o corporativo amortigua eso:

- si ya resolviste `repo.example.com` hace 10 segundos y el TTL es 300, no hace falta salir otra vez;
- si cien hosts preguntan lo mismo, el cache contesta una vez y el resto sale gratis;
- si tu upstream tiene jitter o microcortes, un cache sano reduce impacto visible.

Un **forwarder** actúa como intermediario:

```ascii
Cliente -> Forwarder local -> Resolver upstream -> Internet DNS
```

Un **recursor** puro hace el trabajo completo:

```ascii
Cliente -> Recursor -> Root -> TLD -> Autoritativo
```

Ambos pueden cachear. La diferencia es quién depende de quién para obtener la respuesta inicial.

## Cómo funciona

### Caché positivo y negativo

- **Caché positivo:** guarda respuestas válidas (`A`, `MX`, `NS`, etc.) por el TTL anunciado.
- **Caché negativo:** guarda fallos como `NXDOMAIN` o ciertas ausencias, también por un tiempo controlado.

Esto evita tormentas de consultas sobre nombres inexistentes.

### Forwarding

Con forwarding, el servicio local no interroga root/TLD/autoritativos. Reenvía a uno o más upstreams definidos. Sirve cuando:

- querés una política centralizada;
- la red cliente no necesita recursión completa;
- preferís simplicidad sobre autonomía.

### `dnsmasq`

Fortalezas:

- muy liviano;
- fácil de configurar;
- sirve DNS cache + DHCP + overrides locales;
- excelente en routers, VMs, homelabs, labs de malware, redes chicas.

Limitaciones:

- menos fino que `unbound` para validación, control de recursión y tuning complejo;
- no es la primera opción para un resolver corporativo pesado.

### `unbound`

Fortalezas:

- recursor moderno y serio;
- validación DNSSEC sólida;
- ACLs, views/policies y hardening razonable;
- muy usado como componente de resolución confiable.

Tradeoff:

- más verboso de operar;
- configuración menos "plug and play" que `dnsmasq`.

> [!note]
> `dnsmasq` y `unbound` no son enemigos naturales. Mucha gente usa `dnsmasq` en edge o labs y `unbound` donde necesita un resolver más completo y controlado.

## Comandos / configuración

### Ver quién escucha en el puerto 53

```bash
ss -lntup '( sport = :53 )'
```

Eso te dice si el dueño actual del puerto es `systemd-resolved`, `dnsmasq`, `unbound` u otro proceso.

### `dnsmasq` como forwarder/cache

Archivo:

```text
/etc/dnsmasq.d/lab.conf
```

Ejemplo:

```ini
no-resolv
server=192.168.1.53
server=192.168.1.54
listen-address=127.0.0.1,192.168.1.10
bind-interfaces
cache-size=10000
domain-needed
bogus-priv
local=/lab.example/
address=/git.lab.example/192.168.1.50
```

Aplicar:

```bash
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
dig @127.0.0.1 git.lab.example A
```

### `unbound` como recursor/cache

Archivo:

```text
/etc/unbound/unbound.conf.d/lab.conf
```

Ejemplo:

```ini
server:
    interface: 127.0.0.1
    interface: 192.168.1.10
    access-control: 127.0.0.0/8 allow
    access-control: 192.168.1.0/24 allow
    access-control: 0.0.0.0/0 refuse
    cache-max-ttl: 86400
    cache-min-ttl: 60
    hide-identity: yes
    hide-version: yes
    qname-minimisation: yes
    harden-glue: yes
    harden-dnssec-stripped: yes

forward-zone:
    name: "."
    forward-addr: 192.168.1.53
    forward-addr: 192.168.1.54
```

Aplicar y verificar:

```bash
sudo unbound-checkconf
sudo systemctl restart unbound
sudo systemctl status unbound
dig @127.0.0.1 example.com A
```

### `unbound` como recursor completo

Si no definís `forward-zone` para `"."`, `unbound` puede resolver iterativamente por su cuenta, siempre que tenga reachability y root hints adecuados según tu despliegue.

### Interacción con `systemd-resolved`

Archivo relevante:

```text
/etc/systemd/resolved.conf
```

Estado:

```bash
resolvectl status
ls -l /etc/resolv.conf
```

Punto operativo clave: definí quién es el resolver efectivo del host y hacé que `/etc/resolv.conf` apunte al lugar correcto. Tener dos caches en cascada sin entenderlo complica TTLs, métricas y debugging.

> [!tip]
> Para labs chicos, `dnsmasq` resolviendo local overrides y forwardeando al resolver corporativo suele ser suficiente. Para recursión controlada, DNSSEC y políticas, `unbound` es mejor apuesta.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| El servicio no inicia | Puerto `53` ya ocupado por otro daemon | `ss -lntup '( sport = :53 )'` |
| El host sigue usando otro resolver | `/etc/resolv.conf` o `systemd-resolved` apuntan a otro lado | `resolvectl status`, `cat /etc/resolv.conf` |
| Respuestas viejas persisten | Caché local, del forwarder o del upstream | `dig @127.0.0.1 example.com A +ttlunits` y comparar con autoritativo |
| Solo algunos clientes resuelven | ACLs o `listen-address` mal definidos | revisar config y probar `dig @192.168.1.10 example.com A` desde otro host |
| `SERVFAIL` en `unbound` | Validación DNSSEC, root hints, upstream o política rota | `unbound-checkconf`, logs de `journalctl -u unbound` |

### Chequeos concretos

```bash
journalctl -u dnsmasq --since '15 min ago'
journalctl -u unbound --since '15 min ago'
dig @127.0.0.1 example.com SOA
dig @192.168.1.53 example.com SOA
```

Comparar respuesta local vs upstream te dice rápido si el problema está en tu capa de cache/forwarding o más arriba.

> [!warning]
> Si no deshabilitás la lectura automática de `/etc/resolv.conf` en `dnsmasq` (`no-resolv`) o no definís claramente upstreams, podés crear cadenas opacas de forwarding. Después nadie sabe quién respondió qué.

## Seguridad / ofensiva

Resolveres internos son infraestructura crítica y también superficie interesante de abuso.

### Riesgos defensivos

- **Open resolver** expuesto a internet.
- **Cache poisoning** por software viejo o hardening pobre.
- **DNS rebinding** si no filtrás respuestas privadas según contexto.
- **Logs sensibles** con nombres internos, queries de endpoints o beaconing.
- **Single point of failure** si toda la red depende de un único cache sin redundancia.

### Desde la óptica ofensiva

Un atacante mira si el resolver:

- permite recursión desde segmentos indebidos;
- responde distinto según vista;
- filtra o no dominios maliciosos;
- cachea agresivamente y puede ser usado para ocultar cambios temporales.

También interesa como fuente de inteligencia: consultas frecuentes, dominios internos, nombres de infraestructura, patrones de C2 bloqueados o intentados.

> [!danger]
> Un resolver corporativo con recursión abierta a cualquier origen es útil para terceros como reflector/amplificador y, al mismo tiempo, te expone a abuso reputacional y saturación. Cerrarlo por ACL no es opcional.

### Hardening básico

- ACL estrictas.
- Version hiding y minimal responses donde aplique.
- Separar resolución interna y externa si el modelo lo requiere.
- Monitorear QPS, entropía de queries y picos de `NXDOMAIN`.
- Validar DNSSEC cuando el entorno lo soporte.

## Relacionado

- [[dns-jerarquia-resolucion]]
- [[dig-nslookup-host-drill]]
- [[dns-over-tls-https-doh-dot]]
- [[dns-tunneling-iodine-dnscat2]]

## Referencias

- RFC 1034 - *Domain Names - Concepts and Facilities*
- RFC 1035 - *Domain Names - Implementation and Specification*
- RFC 2308 - *Negative Caching of DNS Queries (DNS NCACHE)*
- RFC 4035 - *Protocol Modifications for the DNS Security Extensions*
- `man 8 dnsmasq`
- `man 8 unbound`
- `man 5 unbound.conf`
- unbound official documentation
- dnsmasq official documentation
- systemd documentation: `systemd-resolved.service(8)` and `resolved.conf(5)`
