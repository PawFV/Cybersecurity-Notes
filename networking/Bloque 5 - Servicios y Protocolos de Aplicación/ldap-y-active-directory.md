---
title: LDAP y Active Directory
tags: [networking, ldap, active-directory, directory-services, identity]
aliases: [LDAP Basico, AD Basico, Directorio Activo]
created: 2026-04-23
difficulty: Intermedio
related:
  - "[[kerberos-basico]]"
  - "[[smb-cifs-y-shares-windows]]"
---

# LDAP y Active Directory

> [!abstract] TL;DR
> - **LDAP** es un protocolo para consultar y modificar servicios de directorio.
> - **Active Directory** es una implementación/directorio de Microsoft que usa LDAP, Kerberos, DNS y otros componentes.
> - LDAP organiza objetos jerárquicamente: usuarios, grupos, equipos, OU y atributos.
> - En AD, LDAP suele convivir con Kerberos: uno consulta identidad y estructura; el otro autentica.

## Concepto

Un directorio es una base orientada a **lecturas frecuentes** y búsquedas por atributos, no a transacciones complejas como una base relacional tradicional. Sirve para responder preguntas como:

- quién es este usuario;
- en qué grupos está;
- qué equipos existen;
- cuál es el mail o el `homeDirectory`;
- qué políticas o permisos aplican.

LDAP define cómo consultar y modificar ese directorio. Active Directory agrega esquema, replication, trust relationships, Group Policy, integración con Kerberos y operación de dominio Windows.

## Cómo funciona

### Estructura jerárquica

Los objetos se ubican en un árbol. Ejemplo:

```ascii
DC=example,DC=local
├── OU=Users
│   ├── CN=Ana Perez
│   └── CN=svc_backup
├── OU=Servers
│   └── CN=FILESRV01
└── CN=Computers
```

Cada entrada tiene un **DN** (Distinguished Name), por ejemplo:

```text
CN=Ana Perez,OU=Users,DC=example,DC=local
```

### Operaciones típicas

LDAP trabaja con operaciones como:

- **bind**: autenticarse contra el directorio;
- **search**: buscar entradas;
- **compare**: comparar atributos;
- **modify**: modificar atributos;
- **add/delete**: alta o baja de entradas.

### LDAP vs LDAPS

- **LDAP** suele asociarse a `389/tcp`.
- **LDAPS** a `636/tcp`.

Además, LDAP puede empezar en claro y luego elevar a TLS usando **StartTLS**.

> [!tip] StartTLS
> No es "otro protocolo". Es la elevación de una sesión LDAP normal a un canal cifrado dentro del mismo puerto lógico.

### Relación con Active Directory

En AD, muchas consultas administrativas y de aplicaciones se hacen por LDAP, mientras la autenticación preferida es Kerberos. Eso explica por qué una app puede "hablar LDAP" para buscar usuarios, pero apoyarse en Kerberos o NTLM para autenticación real.

## Comandos / configuración

Linux / OpenLDAP tools:

```bash
# Búsqueda anónima o autenticada
ldapsearch -x -H ldap://192.168.56.30 -b "DC=example,DC=local" "(objectClass=user)"

# Búsqueda con bind simple
ldapsearch -x -H ldap://dc01.example.local \
  -D "CN=svc_bind,OU=Service Accounts,DC=example,DC=local" \
  -W \
  -b "DC=example,DC=local" \
  "(sAMAccountName=aperez)"

# LDAP sobre TLS
ldapsearch -H ldaps://dc01.example.local -x -b "DC=example,DC=local" "(objectClass=computer)"
```

Windows / PowerShell:

```powershell
# Consultar usuarios del dominio
Get-ADUser -Filter * -SearchBase "OU=Users,DC=example,DC=local"

# Consultar grupos
Get-ADGroup -Filter *

# Consultar un equipo puntual
Get-ADComputer FILESRV01
```

Puertos habituales en AD relacionados:

```text
53/tcp,udp   DNS
88/tcp,udp   Kerberos
389/tcp,udp  LDAP
445/tcp      SMB
636/tcp      LDAPS
3268/tcp     Global Catalog
3269/tcp     Global Catalog over TLS
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| La búsqueda LDAP no devuelve nada | Base DN incorrecta o filtro mal armado. | `ldapsearch` con un filtro más amplio |
| Bind falla pero las credenciales son válidas | Formato de bind DN / UPN incorrecto o política de auth. | Probar `usuario@example.local` vs DN completo |
| AD funciona por IP pero no por nombre | DNS roto; en AD eso impacta fuerte. | `Resolve-DnsName`, `nslookup`, SRV records |
| LDAPS falla con errores de certificado | Certificado inválido o CA no confiable. | Validar certificado del DC |
| App lenta consultando directorio | Filtros amplios, índices pobres o consulta al GC/DC equivocado. | Revisar filtros y alcance de búsqueda |

## Seguridad / ofensiva

### 1. Enumeración de directorio

LDAP es excelente para mapear entorno:

- usuarios;
- grupos;
- equipos;
- OUs;
- descripciones y atributos operativos;
- relaciones útiles para privesc.

En muchos entornos, una cuenta de dominio de bajo privilegio ya puede leer bastante.

### 2. Riesgos comunes

- binds simples sin TLS;
- cuentas de servicio con permisos excesivos;
- exposición de atributos sensibles;
- hardcoded creds en aplicaciones que consultan LDAP.

> [!warning] Simple bind sin cifrado
> Si usás bind simple sobre `389/tcp` sin StartTLS, la contraseña viaja recuperable por cualquier actor con visibilidad de red.

### 3. Relación con Kerberos y SMB

LDAP suele responder "quién existe y qué atributos tiene"; Kerberos autentica; SMB materializa acceso a recursos. En un entorno Windows real, entender los tres en conjunto te da la foto operativa completa.

## Relacionado

- [[kerberos-basico]]
- [[smb-cifs-y-shares-windows]]

## Referencias

- RFC 4511 - *Lightweight Directory Access Protocol (LDAP): The Protocol*
- RFC 4515 - *LDAP: String Representation of Search Filters*
- Microsoft Learn - *Active Directory Domain Services Overview*
- Microsoft Learn - *Get-ADUser*
- OpenLDAP Administrator's Guide
