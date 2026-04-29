---
title: HTB Recon Checklist
tags: [htb, methodology, recon, checklist]
created: 2026-04-29
related:
  - "[[HTB/Web/sql-injection-cheatsheet|SQL Injection Cheatsheet]]"
  - "[[HTB/snippets/one-liners|One-liners]]"
---
# HTB Recon Checklist

> [!abstract] TL;DR
> Primero entender superficie, después profundidad. No saltes a explotar hasta tener puertos, servicios, versiones, rutas web y posibles usuarios.

## Setup

```bash
export IP=10.10.10.10
export HOST=target.htb
mkdir -p scans loot notes
```

## 1. Host y puertos

```bash
ping -c 2 $IP
sudo nmap -p- --min-rate 5000 -oN scans/all-ports.txt $IP
ports=$(grep -oP '\d+(?=/tcp\s+open)' scans/all-ports.txt | paste -sd, -)
sudo nmap -sC -sV -A -p $ports -oN scans/detail.txt $IP
sudo nmap -sU --top-ports 20 -oN scans/udp-top.txt $IP
```

Anotar:

- TTL aproximado;
- puertos abiertos;
- versiones exactas;
- hostnames/domains filtrados;
- servicios raros o custom.

## 2. Web

```bash
whatweb http://$IP
curl -i http://$IP
curl -k -i https://$IP
```

```bash
feroxbuster -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,js,bak,zip,config
```

Si aparece hostname:

```bash
echo "$IP $HOST" | sudo tee -a /etc/hosts
```

Checklist web:

- `robots.txt`, `sitemap.xml`;
- comments HTML / JS;
- login/register/reset;
- uploads;
- parámetros GET/POST;
- vhosts;
- tecnologías y versiones.

## 3. Servicios comunes

```bash
# SMB
smbclient -L //$IP -N
smbmap -H $IP

# FTP
ftp anonymous@$IP

# NFS
showmount -e $IP

# SNMP
snmpwalk -v2c -c public $IP

# LDAP
ldapsearch -x -H ldap://$IP -s base namingcontexts
```

## 4. Credenciales

Buscar:

- usuarios en web, SMB, FTP, LDAP;
- passwords en backups/configs;
- claves SSH;
- hashes;
- reuse entre servicios.

Probar con criterio:

```bash
crackmapexec smb $IP -u users.txt -p passwords.txt --continue-on-success
crackmapexec winrm $IP -u user -p pass
ssh user@$IP
```

## 5. Después del foothold

```bash
whoami; id; hostname
ip a; ip route; ss -tulpen
```

Elegir ruta:

- Linux: [[HTB/methodology/02-linux-privesc|Linux PrivEsc]]
- Windows: [[HTB/methodology/01-windows-privesc|Windows PrivEsc]]
- AD: [[HTB/methodology/03-AD-attack-paths|AD Attack Paths]]
- Pivoting: [[HTB/methodology/04-pivoting-tunneling|Pivoting & Tunneling]]

## Mini checklist final

```text
[ ] Todos los TCP escaneados
[ ] UDP top revisado
[ ] Web fuzz + vhosts
[ ] Servicios comunes enumerados
[ ] Usuarios y credenciales correlacionados
[ ] Foothold estable
[ ] Privesc local iniciada
```
