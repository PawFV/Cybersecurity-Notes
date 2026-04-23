---
tags: [red-team, lateral-movement, pivoting, mitre-attack]
created: 2026-04-21
source: TryHackMe - Lateral Movement and Pivoting
related:
  - "[[Spawning Processes Remotely]]"
  - "[[Moving Laterally Using WMI]]"
  - "[[Use of Alternate Authentication Material]]"
  - "[[Abusing User Behaviour]]"
  - "[[Port Forwarding]]"
---
Una vez obtenido el primer acceso a la red, rara vez el objetivo final está en la máquina comprometida inicial. El **Lateral Movement** es el conjunto de técnicas que permite moverse a través de la red comprometiendo otros hosts hasta alcanzar el objetivo (usualmente el **Domain Controller** o servidores críticos).

---
## Conceptos clave

### Lateral Movement
Movernos de un host comprometido a otro dentro de la misma red utilizando credenciales, tokens o vulnerabilidades obtenidas previamente.

### Pivoting
Usar un host comprometido como **punto de salto** para acceder a redes/subredes a las que originalmente no teníamos visibilidad desde la máquina del atacante.

> [!info] Pivoting suele requerir técnicas de **port forwarding** o túneles para enrutar tráfico a través del host comprometido.

---

## Fases típicas tras la intrusión inicial

1. **Enumeración**: descubrir hosts, servicios, usuarios y privilegios.
2. **Robo de credenciales**: dumping de LSASS, SAM, tickets Kerberos, etc.
3. **Lateral Movement**: ejecutar código en otras máquinas con las credenciales obtenidas.
4. **Pivoting**: abrir túneles para alcanzar segmentos internos.
5. **Escalada de privilegios de dominio**: llegar a Domain Admin.

---

## TTPs principales (MITRE ATT&CK)

| Técnica | ID | Descripción |
|---------|----|-------------|
| Remote Services | T1021 | SMB, RDP, WinRM, SSH |
| Lateral Tool Transfer | T1570 | Copiar herramientas entre hosts |
| Use of Alternate Authentication Material | T1550 | Pass the Hash / Ticket |
| Exploitation of Remote Services | T1210 | Explotar vulnerabilidades remotas |
| Remote Service Session Hijacking | T1563 | RDP/SSH hijacking |

---

## Protocolos y puertos relevantes

| Servicio | Puerto | Uso en lateral movement |
|----------|--------|-------------------------|
| SMB | 445 | PsExec, administrative shares (`C$`, `ADMIN$`) |
| RPC/DCOM | 135 + dinámicos | WMI, DCOM |
| WinRM | 5985/5986 | PowerShell Remoting |
| RDP | 3389 | Acceso gráfico / hijacking |
| Kerberos | 88 | Pass the Ticket |

> [!tip] Casi todas las técnicas de lateral movement en entornos Windows dependen de **SMB + RPC** o **WinRM**. Bloquear estos puertos entre estaciones de trabajo dificulta mucho el movimiento.
