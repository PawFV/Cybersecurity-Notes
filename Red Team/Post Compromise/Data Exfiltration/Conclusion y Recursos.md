---
tags: [red-team, data-exfiltration, summary]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Data Exfiltration Concepts]]"
  - "[[Exfiltration using TCP Socket]]"
  - "[[Exfiltration using SSH]]"
  - "[[Exfiltration using HTTP(S)]]"
  - "[[Exfiltration using ICMP]]"
  - "[[Exfiltration over DNS]]"
  - "[[DNS Tunneling]]"
---
# Conclusión y Resumen de Técnicas

> [!abstract] TL;DR
> La exfiltración de datos no tiene una única solución perfecta. El atacante debe evaluar el entorno (qué puertos salen, si hay proxies web, si hay inspección de tráfico) y elegir el protocolo que ofrezca el mejor balance entre **sigilo** (evadir detección), **velocidad** (volumen de datos por hora) y **fiabilidad** (estabilidad de la conexión). La clave del éxito radica en **mimetizar** la exfiltración con el ruido de fondo habitual de la red.

## Tabla Comparativa de Técnicas

| Técnica / Protocolo | Perfil de Detección (OPSEC) | Velocidad | Casos de Uso Ideales |
|---|---|---|---|
| **TCP Socket (`nc`)** | 🔴 **Muy Alto** (Protocolos no estándar en puertos arbitrarios) | 🟢 Alta | Redes internas mal segmentadas o sin monitoreo de red (ej. entre VLANs OT). |
| **SSH (`tar \| ssh`)** | 🟡 **Medio** (Tráfico cifrado, pero patrón de flujo de volumen alto) | 🟢 Alta | Servidores Linux que tienen permitido administrar otros equipos externos vía SSH. |
| **HTTP(S) POST** | 🟢 **Bajo** (Se mezcla fácilmente con navegación web legítima) | 🟡 Media | Entornos corporativos con salida filtrada por proxies web explícitos. |
| **ICMP (`ping`)** | 🟡 **Medio** (Anomalías en tamaño y timing de paquetes) | 🔴 Baja | Cuando se requiere un Covert Channel lento para C2 o exfiltrar bases de datos pequeñas. |
| **DNS (Base64)** | 🟡 **Medio** (Alta entropía, picos inusuales de consultas DNS) | 🔴 Baja | Entornos súper restrictivos donde sólo sale el tráfico del DNS corporativo interno. |
| **DNS Tunneling (`iodine`)** | 🔴 **Alto** (Volumen extremo de peticiones DNS atípicas) | 🟡 Media | Acceso total tipo VPN cuando el firewall bloquea todo pero permite resolución de nombres. |

## Principios Fundamentales (Takeaways)

Para maximizar las probabilidades de éxito en una operación de exfiltración, se deben seguir estos principios:

1. **Ofuscación Primaria:** Nunca enviar datos en claro. Emplear como mínimo codificaciones (Base64, Hex) o, preferiblemente, cifrado simétrico (AES) antes de enviar la información, para evadir inspecciones pasivas basadas en firmas.
2. **Abuso de la Confianza (Living Off The Land):** En la medida de lo posible, apoyarse en herramientas y protocolos ya presentes y permitidos en el entorno (HTTPS, SSH, DNS, `tar`, `curl`), en lugar de introducir binarios extraños que puedan disparar alertas del EDR.
3. **Mimetismo (Blending In):** En un C2, el canal de comunicación debe imitar patrones humanos y procesos de negocio:
   - Añadir *Jitter* (aleatoriedad en los tiempos de conexión) para evitar un beaconing predecible.
   - Restringir el volumen de envío a horas laborables.
   - En DNS, usar pocos subdominios largos en vez de miles de subdominios cortos para reducir la tasa de consultas por segundo.
4. **Cifrado Fuerte sobre el Transporte:** El uso de HTTPS, DNS over TLS (DoT) o túneles SSH asegura que los dispositivos intermedios (como DPIs o NIDS sin capacidad de descifrado) solo vean metadatos de conexión, ocultando el volumen y naturaleza de la información robada.

## Casos Reales (Ejemplos Históricos)

* **SunTrust Bank (2018):** Un caso de *insider threat* (amenaza interna). Un empleado exfiltró aproximadamente 1.5 millones de registros de clientes (nombres, direcciones, teléfonos y balances bancarios). La información fue robada desde dentro de la red corporativa usando accesos legítimos abusados.
* **Tesla (2018):** Un empleado descontento logró exfiltrar gigabytes de fotografías confidenciales y código fuente relacionado con el sistema operativo de manufactura hacia servidores de terceros no autorizados.
* **Travelex (2020):** Un ataque clásico de Ransomware (Sodinokibi/REvil) que comenzó explotando vulnerabilidades en servidores expuestos (VPNs sin parchear). Los atacantes no solo cifraron los sistemas sino que exfiltraron Información de Identificación Personal (PII) y datos financieros sensibles, usándolos para doble extorsión.

## Herramientas y Recursos Adicionales

* **[LOTS Project (Living Off Trusted Sites)](https://lots-project.com/):** Un repositorio muy valioso de sitios web y servicios legítimos y altamente confiables (como GitHub, Slack, AWS, Pastebin) que pueden ser abusados para exfiltrar datos o alojar infraestructura de C2 sin levantar sospechas (ya que bloquear esos dominios rompería la operatoria del negocio).
* **[Neo-reGeorg](https://github.com/L-codes/Neo-reGeorg):** Herramienta esencial para HTTP Tunneling. Permite crear proxies SOCKS utilizando un script web en servidores vulnerables.
* **[ICMPDoor](https://github.com/krabelize/icmpdoor):** Script en Python (basado en `scapy`) para establecer reverse shells y canales de C2 puramente sobre paquetes ICMP.
* **[iodine](https://github.com/yarrick/iodine):** La utilidad por excelencia para IP over DNS tunneling, permitiendo encapsular cualquier tipo de tráfico dentro de peticiones de resolución de nombres.
* **[nping (Nmap Suite)](https://nmap.org/nping/):** Utilidad para la generación de paquetes de red a medida (TCP, UDP, ICMP), excelente para inyectar datos manualmente en campos específicos de un protocolo.

---

> [!warning] La Realidad del Blue Team
> Ningún canal de exfiltración es verdaderamente "invisible". SOCs y sistemas IDS/IPS modernos (Zeek, Suricata) están diseñados para detectar exfiltraciones buscando **anomalías estadísticas de volumen**, **entropía en parámetros**, **inversiones de ratio de tráfico** (un host interno enviando de repente muchísima más información al exterior de la que recibe) y **beacons temporales**. Siempre existe un compromiso estricto (trade-off) entre la velocidad, el sigilo y la fiabilidad.