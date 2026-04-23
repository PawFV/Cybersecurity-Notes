---
title: SMB CIFS y Shares en Windows
tags: [networking, smb, cifs, windows, fileshares]
aliases: [SMB Basico, Windows Shares, CIFS]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[ldap-y-active-directory]]"
  - "[[kerberos-basico]]"
---

# SMB CIFS y Shares en Windows

> [!abstract] TL;DR
> - **SMB** es el protocolo de compartición de archivos, impresoras y recursos remotos predominante en entornos Windows.
> - Hoy se habla de **SMB 2/3**; **CIFS** suele usarse como nombre histórico para SMB1.
> - Un share (`\\server\share`) expone recursos con permisos de share y permisos NTFS, que se combinan.
> - SMB es central para administración Windows, pero también para movimiento lateral, enumeración y abuso de credenciales.

## Concepto

SMB (Server Message Block) permite acceder a recursos remotos como si fuesen parte de la máquina local: carpetas, impresoras, named pipes y algunos mecanismos RPC. En Windows es parte del tejido operativo del dominio.

La ruta UNC típica es:

```text
\\filesrv01.example.local\Public
```

Eso representa:

- host remoto: `filesrv01.example.local`
- recurso compartido: `Public`

No alcanza con que exista el share. El acceso efectivo depende de dos capas:

1. **Share permissions**
2. **NTFS permissions**

El permiso real termina siendo la intersección práctica entre ambas.

## Cómo funciona

### Puertos y transporte

SMB moderno opera principalmente sobre:

- `445/tcp`: SMB directo sobre TCP.
- `139/tcp`: NetBIOS Session Service, histórico/legacy.

Antes había más dependencia de NetBIOS (`137/udp`, `138/udp`, `139/tcp`), pero en redes modernas el foco real es `445/tcp`.

### Negociación

Cuando el cliente conecta:

1. negocia dialecto SMB;
2. autentica usuario/máquina;
3. establece sesión;
4. se conecta al tree/share;
5. opera archivos, pipes o RPCs.

```ascii
Cliente                                  Servidor
   | TCP 445 ------------------------------> |
   | SMB Negotiate Request                  |
   |--------------------------------------->|
   | SMB Negotiate Response                 |
   |<---------------------------------------|
   | Session Setup (NTLM / Kerberos)        |
   |--------------------------------------->|
   | Tree Connect \\server\share            |
   |--------------------------------------->|
   | Read / Write / Query / RPC             |
```

### Autenticación

En workgroup y escenarios simples aparece **NTLM**. En dominio, lo esperado es **Kerberos**, con NTLM como fallback en muchos casos. Esa diferencia importa muchísimo para hardening y detección.

> [!tip] SMB signing
> SMB signing agrega integridad al tráfico. No cifra por sí solo en todas las versiones, pero dificulta ciertos MITM y relay según el contexto.

### SMB 1 vs 2/3

- **SMB1 / CIFS**: viejo, verboso, inseguro y desaconsejado.
- **SMB2**: reduce chattyness y mejora rendimiento.
- **SMB3**: agrega mejoras de seguridad, multichannel y cifrado.

> [!warning] SMB1
> Mantener SMB1 habilitado amplía mucho la superficie de ataque. En redes actuales debería estar desactivado salvo dependencia muy justificada y compensada.

## Comandos / configuración

Windows:

```powershell
# Ver shares locales
Get-SmbShare

# Ver sesiones SMB activas
Get-SmbSession

# Mapear un share
New-PSDrive -Name Z -PSProvider FileSystem -Root "\\filesrv01.example.local\Public" -Persist

# Enumerar conexiones actuales
net use
```

Linux:

```bash
# Listar shares expuestos
smbclient -L //192.168.56.20 -U usuario

# Conectar a un share
smbclient //192.168.56.20/Public -U usuario

# Montar un share SMB
sudo mount -t cifs //192.168.56.20/Public /mnt/public -o username=usuario,domain=EXAMPLE
```

Configuración Samba mínima en Linux:

```ini
[Public]
    path = /srv/samba/public
    browseable = yes
    read only = no
    guest ok = no
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| `System error 53` / no encuentra la ruta | DNS, reachability o firewall a `445/tcp`. | `Test-NetConnection filesrv01.example.local -Port 445` |
| `Access is denied` | Permisos NTFS/share insuficientes o credenciales inválidas. | Revisar ACLs y `Get-SmbSession` |
| Conecta por IP pero no por nombre | Problema de DNS o resolución NetBIOS. | `Resolve-DnsName filesrv01.example.local` |
| Transferencias muy lentas | Signing, latencia, MTU o WAN desfavorable. | `ping`, `pathping`, counters SMB |
| Fallan autenticaciones a recursos de dominio | Desfase horario o problema Kerberos. | `klist`, sincronización NTP, logs de seguridad |

## Seguridad / ofensiva

### 1. Enumeración

SMB revela mucho valor operativo:

- shares accesibles;
- nombres de host;
- política de signing;
- sesiones activas;
- exposición de pipes/RPC.

```bash
nmap --script smb-protocols,smb2-security-mode -p 445 192.168.56.20
```

### 2. Movimiento lateral

Históricamente SMB fue vector clave para:

- copiar payloads;
- ejecutar servicios remotos;
- PsExec-style operations;
- despliegue lateral con credenciales válidas.

### 3. Relay y hardening

Si hay NTLM, signing ausente y servicios susceptibles, aparecen ataques de relay. Por eso defensivamente conviene:

- exigir SMB signing donde sea viable;
- deshabilitar SMB1;
- limitar acceso administrativo a shares ocultos;
- monitorear acceso a `ADMIN$`, `C$`, `IPC$`.

> [!danger] Shares administrativos
> `ADMIN$`, `C$` y `IPC$` son normales en Windows, pero para un atacante con credenciales válidas son una autopista operativa.

### 4. Cifrado SMB

SMB 3 puede cifrar tráfico extremo a extremo entre cliente y servidor compatible. En segmentos inseguros o redes internas de baja confianza, vale la pena considerarlo.

## Relacionado

- [[ldap-y-active-directory]]
- [[kerberos-basico]]

## Referencias

- Microsoft Learn - *SMB overview*
- Microsoft Learn - *SMB security enhancements*
- Microsoft Learn - *Get-SmbShare* / *Get-SmbSession*
- Samba Documentation
- MS-SMB2 - *Server Message Block (SMB) Protocol Versions 2 and 3*
