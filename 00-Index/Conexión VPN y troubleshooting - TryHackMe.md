---
tags:
  - tryhackme
  - lab
  - networking
  - vpn
  - troubleshooting
created: 2026-04-19
---

# TryHackMe — Conexión VPN y troubleshooting

> [!warning] Trampa común
> Status "Connected" en la web de THM = túnel OpenVPN up en algún lado. NO significa que tu VM lo use.

## Por qué la VM no hereda VPN del host
VirtualBox da adapter virtual con stack de red propia. `tun0` de OpenVPN en Windows vive en kernel del host → **la VM lo ve como máquina física aparte**.

→ Hacer routing manual (IP forwarding + rutas estáticas) = quilombo innecesario.

→ **Siempre correr OpenVPN dentro de Kali.**

## Troubleshooting en orden

### 1. ¿Está el túnel en la VM?
```bash
ip a | grep -A2 tun0
ip route | grep 10\.
```
Debe aparecer interfaz `tun0` con IP del panel THM + rutas a rangos 10.x.

Si no → matar OpenVPN en Windows, levantar en Kali:
```bash
sudo openvpn /ruta/archivo.ovpn
```

### 2. Reachability
```bash
ping -c 3 <TARGET_IP>
```
Sin respuesta → 99% **máquina no iniciada** en THM (botón `Start Machine`).

### 3. Puerto abierto
```bash
nmap -Pn -p 3389 <TARGET_IP>
```

### 4. RDP con flags útiles
```bash
xfreerdp /v:<IP> /u:<user> /cert:ignore /dynamic-resolution
```

## Pasar archivos Host → VM

### Shared Folder (prolija)
1. VM apagada → VBox → Settings → Shared Folders
2. Auto-mount + Permanent
3. En Kali: instalar guest additions, `usermod -aG vboxsf $USER`, reboot
4. `cp /media/sf_share/archivo ~/`

### Drag & drop
Devices → Shared Clipboard + Drag and Drop: **Bidirectional**

### Cochina pero instantánea
Descargar `.ovpn` directamente desde browser de Kali. Cero config.