---
title: UDP vs TCP (Cuándo usar cuál)
tags: [networking, l4, tcp, udp, escaneos]
aliases: [TCP vs UDP, Protocolos de Transporte]
created: 2026-04-23
difficulty: Básico
related:
  - "[[tcp-estados-y-handshake]]"
  - "[[dns-jerarquia-resolucion]]"
---

# UDP vs TCP (Cuándo usar cuál)

> [!abstract] TL;DR
> - **TCP** garantiza entrega, orden y control de congestión a costa de overhead (headers pesados, latencia por handshakes).
> - **UDP** es de entrega en el "mejor esfuerzo". Tira el paquete a la red y se olvida. Si se pierde, problema de la aplicación (L7).
> - **Cuándo TCP:** Web (HTTP/S), Email (SMTP), Transferencia de Archivos (SSH, FTP), Bases de Datos. Todo lo que no puede tener un byte corrupto.
> - **Cuándo UDP:** Streaming de video/audio, VoIP, Juegos Online, DNS (consultas rápidas), NTP (hora). Todo donde el tiempo real importa más que perder un frame.

## Concepto

La analogía obligada:
- **TCP (Transmission Control Protocol)** es mandar una carta certificada con acuse de recibo. El cartero te asegura que la carta llegó en perfectas condiciones y en el orden en que las enviaste. Si no llega, te avisa y la mandás de nuevo.
- **UDP (User Datagram Protocol)** es subirte a un balcón y tirar volantes a la calle. Esperás que alguien los agarre, pero no tenés forma de saber quién lo hizo, ni si el viento se llevó la mitad, ni en qué orden los leyeron.

## Cómo funciona

### El peso del Header
La diferencia de filosofía se nota en la estructura del paquete en L4.
- **Header TCP:** 20 a 60 bytes. Contiene Seq Nums, Ack Nums, Flags (SYN, ACK, FIN), Window Size (para control de flujo).
- **Header UDP:** Solo 8 bytes. Contiene Puerto Origen, Puerto Destino, Longitud y un Checksum opcional. Nada más.

### Sin conexiones (Connectionless)
Como UDP no tiene concepto de "Conexión", no hay Handshake. Si mandás un paquete UDP al puerto 53 (DNS), sale instantáneamente. Esto reduce la latencia a la mitad comparado con TCP (que necesita un viaje de ida y vuelta antes de mandar el primer byte útil).

## Comandos / configuración

Monitorear tráfico UDP es fundamentalmente distinto a TCP porque el SO no mantiene "estados" fijos de sesión (ESTABLISHED, CLOSE_WAIT). Solo sabe si un proceso tiene el puerto abierto para recibir.

```bash
# ========================================
# Ver puertos a la escucha
# ========================================
ss -uan      # Mostrar sockets UDP (-u), sin resolver nombres (-n) y todos (-a)
ss -tuan     # Mostrar tanto TCP (-t) como UDP (-u)

# ========================================
# Interacción básica con Netcat
# ========================================
# TCP (Por defecto)
nc -lvp 8080         # Escucha
nc 192.168.1.10 8080 # Conecta

# UDP (Bandera -u)
nc -ulvp 8080        # Escucha UDP
nc -u 192.168.1.10 8080 # Manda datagramas UDP (No hay "conexión" real)
```

## Troubleshooting

| Síntoma | Causa probable | Comando de diagnóstico |
|---------|----------------|------------------------|
| Un puerto UDP parece abierto pero el servicio no responde | No hay *Connection Refused* en UDP. Si un firewall dropea el paquete, simplemente hay timeout. El puerto se verá como `open|filtered` en Nmap. | `nmap -sU -p <puerto> <IP>`. Mandar tráfico legítimo del protocolo (ej. una query DNS real) para forzar una respuesta. |
| Video pixelado / Voz robótica en VoIP | Los paquetes UDP están llegando desordenados o perdiéndose (Jitter / Packet Loss), pero como no hay retransmisión (como en TCP), la app intenta reproducir los huecos. | `ping` continuo o `mtr` para medir pérdida de paquetes en L3. |

## Seguridad / ofensiva

Desde la perspectiva del Red Team, UDP es una caja de sorpresas. Como los protocolos UDP no verifican quién les manda el paquete (no hay handshake que compruebe la IP origen), son el vector ideal para dos grandes ataques:

### 1. IP Spoofing y Ataques de Amplificación (DDoS)
Si enviás un paquete UDP diminuto (ej. 60 bytes) a un servidor DNS o NTP mal configurado, y **falsificás tu IP de origen (Spoofing)** para que parezca la IP de tu víctima... el servidor DNS le responderá *a la víctima*. 
Si le pediste al DNS una respuesta masiva (ej. "dame todos los registros de esta zona", que pesan 3000 bytes), acabás de **amplificar** tu ataque x50. Con una botnet pequeña, podés ahogar la conexión de cualquier empresa.

*Servicios comúnmente abusados:* DNS, NTP, Memcached, SSDP.

### 2. Escaneos UDP (`nmap -sU`)
Escanear puertos UDP es un dolor de cabeza.
- Si el puerto está **cerrado**, el SO del destino *debería* responder con un paquete ICMP "Port Unreachable" (Tipo 3, Código 3).
- Si el puerto está **abierto**, el servicio recibe tu paquete basura de Nmap, pero como no lo entiende (ej. le mandás basura a un servidor SNMP), lo ignora silenciosamente.
- Si hay un **firewall** en el medio y descarta el paquete (Drop), tampoco recibís respuesta.

Para Nmap, el silencio del puerto abierto y el silencio del firewall son idénticos. Por eso Nmap los marca como `open|filtered`. Toma muchísimo tiempo (a veces horas) escanear los 65535 puertos UDP de forma confiable.

> [!tip] Estrategia de escaneo
> Nunca escanees todos los puertos UDP al mismo tiempo que los TCP. Tirá el escaneo TCP normal (`-sS`) y lanzá en paralelo un escaneo UDP solo a los puertos top (`nmap -sU --top-ports 100`).

### 3. Exfiltración / C2 sobre UDP
Dado que el tráfico UDP a menudo se tolera mejor hacia el exterior para servicios como DNS (puerto 53) y NTP (puerto 123), un atacante puede encapsular tráfico TCP dentro de UDP (ej. usando VPNs como WireGuard en puertos no estándar, o herramientas como `iodine` para DNS tunneling).

## Relacionado
- [[tcp-estados-y-handshake]] (El contracara L4)
- [[dns-tunneling-iodine-dnscat2]] (Cómo abusar del tráfico UDP 53)

## Referencias
- RFC 768 - *User Datagram Protocol*
- Man pages: `man nc`, `man ss`
