---
tags: [red-team, data-exfiltration, dns, lab-setup]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Exfiltration over DNS]]"
  - "[[DNS Tunneling]]"
---
# DNS: Setup e Infraestructura

> [!abstract] TL;DR
> Antes de poder exfiltrar datos sobre DNS, el atacante necesita configurar infraestructura que le permita **recibir** las consultas DNS generadas por la víctima. Esto implica poseer un dominio y configurar registros específicos (`A` y `NS`) para convertir la máquina del atacante en el servidor DNS Autoritativo de un subdominio particular.

## El problema a resolver

Para que un ataque sobre DNS funcione, la víctima no se comunica directamente con la IP del atacante. En cambio, hace una consulta DNS normal a su servidor local (ej. el DNS corporativo o el de su ISP). Ese servidor resuelve la cadena de jerarquías hasta que se da cuenta de que el atacante es el "dueño" (autoritativo) de ese dominio, y le **re-envía** la consulta original.

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   VÍCTIMA   │ ────▶ │ DNS INTERNO │ ────▶ │   ATACANTE  │
│             │       │ (Resolver)  │       │(Name Server)│
└─────────────┘       └─────────────┘       └─────────────┘
  1. "Quiero ver       2. "Yo no sé.         3. "Acá estoy,
      sub.atck.com"        Le pregunto al      recibo la info"
                           dueño de atck.com"
```

Si no configuramos correctamente el atacante como un Name Server (NS), la consulta se perderá en internet.
## Requisitos de infraestructura

Para levantar un "Name Server" que intercepte nuestra exfiltración necesitamos dos piezas clave en el panel de control de nuestro dominio (ej. en GoDaddy, Cloudflare o Route53):

### 1. El registro `A` (Address Record)

Primero, debemos decirle a Internet cuál es la dirección IP de nuestro servidor atacante. Creamos un registro `A` para un subdominio específico (por ejemplo, `ns1`).

```
Type: A   Subdomain: ns1   Value: <ATTACKBOX_IP>
```
*Traducción: La IP del servidor "ns1.tunnel.com" es la máquina del atacante.*

### 2. El registro `NS` (Name Server Record)

Luego, debemos delegar la autoridad de otro subdominio (por ejemplo, `t1`) hacia el registro `A` que acabamos de crear.

```
Type: NS  Subdomain: t1     Value: ns1.tunnel.com
```
*Traducción: Todo lo que termine en ".t1.tunnel.com" debe ser resuelto preguntándole a "ns1.tunnel.com" (que es el atacante).*

> [!tip]
> El valor del registro NS debe ser siempre el **FQDN (Fully Qualified Domain Name) completo**, no solo el subdominio. Debe decir `ns1.tunnel.com`, no solo `ns1`.

Con esta configuración, cualquier consulta hacia `*.t1.tunnel.com` (donde `*` será nuestra data exfiltrada) terminará golpeando el puerto 53 (UDP) de la máquina atacante.

## Ejemplo: Nameserver preconfigurado en el Laboratorio

En entornos controlados o laboratorios (como TryHackMe), esta infraestructura ya suele estar simulada. Por ejemplo, existe un dominio `tunnel.com` y el host atacante interno (`attacker.thm.com = 172.20.0.200`) ya está configurado así:

| Registro (Record) | Tipo (Type) | Valor (Value) | Propósito |
|---|---|---|---|
| `attNS.tunnel.com` | A | `172.20.0.200` | Asocia el nombre del Name Server con la IP del atacante. |
| `att.tunnel.com` | NS | `attNS.tunnel.com` | Delega el tráfico de `*.att.tunnel.com` hacia el atacante. |

## Forzar el uso de un DNS (Si es necesario)

Si por alguna razón la máquina atacante (AttackBox) o la víctima no pueden resolver el DNS del laboratorio correctamente, es posible que tengamos que forzarlas a usar un servidor DNS específico editando su configuración de red.

### Cambiar DNS en Linux mediante Netplan

Si el sistema usa Netplan, editamos el archivo YAML correspondiente:

```shell
root@AttackBox:~# nano /etc/netplan/aws-vmimport-netplan.yaml
```

```yaml
network:
    ethernets:
        eth0:
            dhcp4: true
            optional: false
            nameservers:
               search: [tunnel.com]
               addresses: [MACHINE_IP]
        ens5:
            dhcp4: true
            optional: false
    version: 2
```

Aplicamos la configuración para que el sistema empiece a enrutar consultas hacia esa IP:

```shell
netplan apply
```

> [!warning]
> Netplan a veces es caprichoso. Puede que sea necesario ejecutar `netplan apply` dos veces seguidas para que el sistema operativo purgue la caché y tome los cambios correctamente.

## Verificar que el DNS funciona

Antes de empezar a inyectar comandos o exfiltrar archivos, siempre debemos probar que la resolución DNS funciona correctamente de extremo a extremo. Para eso usamos `dig` y `ping`.

```shell
# Consultar al DNS interno si conoce la IP de test.thm.com
thm@jump-box:~$ dig +short test.thm.com
127.0.0.1

# Confirmar conectividad base
thm@jump-box:~$ ping test.thm.com -c 1
PING test.thm.com (127.0.0.1) 56(84) bytes of data.
```

Si `test.thm.com` y `test.tunnel.com` resuelven correctamente (en este caso a localhost simulando un servicio), significa que el enrutamiento DNS está operativo y estamos listos para la técnica de exfiltración real.

## Preguntas y Respuestas

- **Una vez la config DNS funcione, resolvé `flag.thm.com`. ¿Cuál es la IP?**
  → Ejecutar `dig +short flag.thm.com`. Devolverá la IP asignada (generalmente un rango interno estilo `172.20.0.120`).