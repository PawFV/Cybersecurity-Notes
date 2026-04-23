---
title: resolv.conf y systemd-resolved
tags: [networking, dns, linux, systemd, resolver]
aliases: [resolv.conf, systemd-resolved, DNS en Linux]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[dns-jerarquia-resolucion]]"
  - "[[systemd-networkd-vs-networkmanager]]"
  - "[[netplan]]"
---

# resolv.conf y systemd-resolved

> [!abstract] TL;DR
> - `resolv.conf` define cómo el resolver del sistema consulta DNS, pero en Linux moderno muchas veces es un archivo generado o un symlink.
> - `systemd-resolved` actúa como stub resolver local, caché, coordinador de DNS por interfaz y punto de integración con `networkd` o `NetworkManager`.
> - Si tocás `/etc/resolv.conf` a mano sin entender quién lo gestiona, probablemente tu cambio dure poco.
> - Para troubleshooting de nombres, hay que separar resolución del sistema, reachability a los nameservers y comportamiento de aplicaciones.

## Concepto

La resolución DNS en Linux suele tener dos capas:

1. **Resolver de libc / NSS**: lo que usan muchas aplicaciones al pedir un hostname.
2. **Servicio resolvedor local o archivo de configuración**: hacia dónde salen las consultas y con qué política.

Históricamente eso se reducía a `/etc/resolv.conf`. Hoy la película es más compleja:

- `systemd-resolved` puede escuchar en `127.0.0.53`
- `NetworkManager` puede empujar DNS por interfaz
- DHCP puede actualizar nameservers
- `resolv.conf` puede ser symlink a un archivo manejado dinámicamente

```ascii
Aplicación -> libc / NSS -> 127.0.0.53 (systemd-resolved) -> DNS upstream
                               |
                               +-> cache
                               +-> DNS por interfaz
                               +-> search domains
```

> [!note]
> "No resuelve DNS" no es un diagnóstico. Puede fallar el stub local, el nameserver upstream, la ruta hacia ese nameserver, la política NSS o la propia aplicación.

## Cómo funciona

### `resolv.conf`

Suele contener directivas como:

- `nameserver`
- `search`
- `options`

Ejemplo clásico:

```conf
nameserver 192.0.2.53
nameserver 192.0.2.54
search corp.example
options timeout:2 attempts:2
```

### `systemd-resolved`

Puede actuar como:

- stub resolver local
- caché DNS
- coordinador de DNS por enlace
- resolvedor con soporte para DNSSEC y DNS over TLS en ciertos escenarios

Consulta estado con:

```bash
resolvectl status
resolvectl query example.com
systemctl status systemd-resolved
```

En muchos sistemas:

```bash
ls -l /etc/resolv.conf
```

devuelve un symlink hacia uno de estos destinos:

- `/run/systemd/resolve/stub-resolv.conf`
- `/run/systemd/resolve/resolv.conf`

La diferencia importa: uno apunta al stub local, el otro lista upstreams directamente.

## Comandos / configuración

Inspección básica:

```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
resolvectl status
resolvectl dns
resolvectl domain
getent hosts example.com
dig @192.0.2.53 example.com
```

Consulta puntual por interfaz:

```bash
resolvectl query internal.example --interface eth0
```

Configurar DNS persistente depende del gestor:

- `systemd-networkd`: en `.network`
- `NetworkManager`: con `nmcli`
- `netplan`: en YAML bajo `nameservers`

Ejemplo `systemd-networkd`:

```ini
[Match]
Name=eth0

[Network]
DNS=192.0.2.53
Domains=corp.example
```

> [!tip]
> Para validar si el problema es DNS o routing, probá ambos caminos: `getent hosts example.com` y `dig @192.0.2.53 example.com`. El primero prueba la cadena del sistema; el segundo, el servidor DNS elegido.

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| `ping 8.8.8.8` funciona pero `ping example.com` no | Falla de resolución, no de reachability IP | `getent hosts example.com`, `resolvectl status` |
| Editás `resolv.conf` y se revierte | Archivo gestionado por `resolved`, DHCP o NM | `ls -l /etc/resolv.conf`, revisar servicio gestor |
| Una VPN resuelve dominios internos solo a veces | DNS por interfaz o split DNS mal aplicado | `resolvectl status`, `nmcli dev show` |
| `dig` funciona pero la app no | NSS, caché, stub local o app usando su propio resolver | `getent`, `strace` si corresponde, revisar app |
| Respuestas lentas | nameserver caído, timeout alto o fallback raro | `resolvectl query`, `tcpdump -ni eth0 port 53` |

> [!warning]
> `dig` no representa siempre lo que ve la aplicación. Muchas apps usan libc/NSS; otras implementan su propio stub resolver. No mezcles ambos resultados sin contexto.

## Seguridad / ofensiva

### Riesgos ofensivos

- Cambiar DNS puede redirigir tráfico, facilitar phishing interno o exfiltración vía dominios controlados.
- Search domains mal gestionados amplían superficie para ataques de resolución oportunista.
- Un resolver local con cache poisoning o configuración laxa puede afectar a muchos procesos del host.

### Visión Red Team

En footholds Linux, revisar resolución sirve para:

- descubrir dominios internos
- identificar DNS corporativos
- inferir segmentación o presencia de VPN
- planear tunneling o resolución de infraestructura interna

```bash
resolvectl status
cat /etc/resolv.conf
```

### Visión defensiva

- Minimizar edición manual de `resolv.conf`.
- Establecer un único owner del DNS del host.
- Auditar search domains innecesarios.
- Monitorear cambios en configuración de resolvedor y en queries anómalas.

> [!danger]
> Un atacante que controla DNS no solo "rompe nombres": puede manipular trust paths, resolver infraestructura falsa y degradar controles basados en FQDN. El impacto suele ser mayor de lo que parece.

## Relacionado

- [[dns-jerarquia-resolucion]] (fundamentos DNS)
- [[systemd-networkd-vs-networkmanager]] (quién inyecta DNS)
- [[netplan]] (config persistente en Ubuntu)

## Referencias

- `man resolv.conf`
- `man systemd-resolved`
- `man resolvectl`
- RFC 1034 - *Domain Names - Concepts and Facilities*
- RFC 1035 - *Domain Names - Implementation and Specification*
- [systemd-resolved documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html)
- [systemd-resolved and resolv.conf](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html#/etc/resolv.conf)
