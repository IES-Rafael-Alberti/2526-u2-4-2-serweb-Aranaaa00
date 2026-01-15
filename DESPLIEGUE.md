# DESPLIEGUE — Evidencias y respuestas

Este documento recopila todas las evidencias y respuestas de la practica.

---

## Parte 1 — Evidencias minimas

### Fase 1: Instalacion y configuracion

1) Servicio Nginx activo
- Que demuestra: El servidor Nginx esta corriendo correctamente dentro del contenedor Docker
- Comando: `docker compose ps`
- Evidencia: ![Nginx activo](./evidencias/01-nginx-activo.png)

2) Configuracion cargada
- Que demuestra: El archivo de configuracion personalizado (default.conf) esta correctamente montado en el contenedor
- Comando: `docker exec nginx-web ls -la /etc/nginx/conf.d/`
- Evidencia: ![Configuracion cargada](./evidencias/02-configuracion-cargada.png)

3) Resolucion de nombres
- Que demuestra: El archivo hosts de Windows esta configurado para resolver nombres personalizados (127.0.0.1 miweb.local)
- Evidencia: ![Resolucion de nombres](./evidencias/03-resolucion-nombres.png)

4) Contenido Web
- Que demuestra: La pagina web personalizada de Cloud Academy se muestra correctamente (no es la pagina por defecto de Nginx)
- Evidencia: ![Contenido Web](./evidencias/03-resolucion-nombres.png)

### Fase 2: Transferencia SFTP (Filezilla)

5) Conexion SFTP exitosa
- Que demuestra: FileZilla se conecta correctamente al servidor SFTP usando localhost:2222 con credenciales usuario/password
- Evidencia: ![Conexion SFTP](./evidencias/05-conexion-sftp.png)

6) Permisos de escritura
- Que demuestra: Los archivos se pueden subir por SFTP sin errores de permisos
- Evidencia: ![Permisos escritura](./evidencias/06-permiso-escritura.png)

### Fase 3: Infraestructura Docker

7) Contenedores activos
- Que demuestra: Ambos contenedores (nginx-web y sftp-server) estan corriendo simultaneamente con sus puertos mapeados
- Comando: `docker compose ps`
- Evidencia: ![Contenedores activos](./evidencias/01-nginx-activo.png)

8) Persistencia (Volumen compartido)
- Que demuestra: Los archivos subidos por SFTP aparecen automaticamente en la web gracias al volumen compartido shared-data
- Evidencia: ![Persistencia volumen](./evidencias/07-persistencia-volumen.png)

9) Despliegue multi-sitio
- Que demuestra: El servidor web sirve multiples sitios - la web principal en / y el reloj en /reloj
- Evidencia: ![Reloj funcionando](./evidencias/08-multisitio-reloj.png)

### Fase 4: Seguridad HTTPS

10) Cifrado SSL
- Que demuestra: El servidor responde por HTTPS con certificados SSL autofirmados correctamente configurados
- Evidencia: ![Cifrado SSL](./evidencias/09-cifrado-ssl.png)

11) Redireccion forzada
- Que demuestra: Las peticiones HTTP (puerto 8080) se redirigen automaticamente a HTTPS (puerto 8443) con codigo 301
- Evidencia: ![Redireccion HTTPS](./evidencias/10-redireccion-https.png)

---

## Parte 2 — Evaluacion RA2 (a–j)

### a) Parametros de administracion

Para ver los parametros de administracion mas importantes de Nginx, he buscado dentro del contenedor en el archivo `/etc/nginx/nginx.conf`:

```bash
docker compose exec web sh -c "grep -nE 'worker_processes|worker_connections|access_log|error_log|gzip|include|keepalive_timeout' /etc/nginx/nginx.conf"
```

El resultado muestra donde estan las directivas:

```
3:worker_processes  auto;
5:error_log  /var/log/nginx/error.log notice;
10:    worker_connections  1024;
15:    include       /etc/nginx/mime.types;
22:    access_log  /var/log/nginx/access.log  main;
27:    keepalive_timeout  65;
29:    #gzip  on;
31:    include /etc/nginx/conf.d/*.conf;
```

Evidencia: ![Directivas nginx.conf](./evidencias/11-grep-nginxconf.png)

Segun lo que he aprendido, cada directiva hace lo siguiente:

- **worker_processes**: Cuantos procesos crea Nginx para atender peticiones
- **error_log**: Donde guarda los errores
- **worker_connections**: Cuantas conexiones puede manejar cada worker
- **include**: Carga otros archivos de configuracion
- **access_log**: Donde guarda el registro de visitas
- **keepalive_timeout**: Cuanto tiempo mantiene una conexion abierta sin actividad
- **gzip**: Para comprimir las respuestas (esta comentado)

#### Cambio aplicado

He modificado el `keepalive_timeout` de 65 a 30 segundos. Este cambio lo he hecho en mi archivo `default.conf` que se monta en el contenedor, asi no toco el nginx.conf original:

```nginx
keepalive_timeout 30;
```

Despues he validado que la configuracion es correcta:

```bash
docker compose exec web nginx -t
```

Evidencia: ![Validacion nginx -t](./evidencias/12-nginx-t.png)

Y finalmente he recargado Nginx para aplicar los cambios:

```bash
docker compose exec web nginx -s reload
```

Evidencia: ![Recarga nginx -s reload](./evidencias/13-reload.png)

### b) Ampliacion de funcionalidad + modulo investigado

He elegido la opcion B2: cabeceras de seguridad.

#### Cabeceras de seguridad

He añadido tres cabeceras de seguridad en el bloque server HTTPS de mi `default.conf`:

```nginx
# Cabeceras de seguridad
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options DENY;
add_header Content-Security-Policy "default-src 'self'";
```

Que significa cada una:

- **X-Content-Type-Options: nosniff** → Evita que el navegador intente adivinar el tipo de archivo. Si digo que es HTML, es HTML y punto.

- **X-Frame-Options: DENY** → Prohibe que mi web se cargue dentro de un iframe de otra pagina. Esto evita ataques de clickjacking.

- **Content-Security-Policy: default-src 'self'** → Solo permite cargar recursos (scripts, imagenes, etc) desde mi propio dominio.

Evidencia: ![Cabeceras en default.conf](./evidencias/14-defaultconf-headers.png)

Despues valide la configuracion:

```bash
docker compose exec web nginx -t
```

Evidencia: ![Validacion nginx -t](./evidencias/12-nginx-t.png)

Y comprobe con curl que las cabeceras aparecen en la respuesta HTTPS:

```bash
curl.exe -I -k https://localhost:8443/
```

El resultado muestra las tres cabeceras:
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
```

Evidencia: ![Cabeceras en respuesta](./evidencias/15-curl-https-headers.png)

#### Modulo investigado: ngx_http_headers_more_module (su documentación esta en ingles pero me parecio bastante interesante)

Este modulo permite modificar las cabeceras HTTP de forma mas avanzada que el `add_header` normal de Nginx.

**Para que sirve:**
- Permite añadir, modificar o eliminar cualquier cabecera de peticion o respuesta
- Puede modificar cabeceras que el add_header normal no puede tocar (como Server o Date)
- Util para ocultar la version de Nginx por seguridad

**Como se instala:**
En Nginx normal hay que compilarlo desde el codigo fuente o instalarlo como modulo dinamico. En algunas distribuciones viene en el paquete `nginx-extras`:

```bash
# En Debian/Ubuntu
apt install nginx-extras

# O compilando desde fuente
./configure --add-module=/path/to/headers-more-nginx-module
```

Despues se carga con:
```nginx
load_module modules/ngx_http_headers_more_filter_module.so;
```

**Fuente consultada:**
- Repositorio oficial del modulo: https://github.com/openresty/headers-more-nginx-module

### c) Sitios virtuales / multi-sitio

En este proyecto tengo configurado un multi-sitio por path, es decir, varias webs dentro del mismo servidor pero en rutas diferentes:

- La web principal esta en `/` → https://localhost:8443/
- La web del reloj esta en `/reloj` → https://localhost:8443/reloj

Evidencia web principal: ![Web principal](./evidencias/16-root.png)

Evidencia web reloj: ![Web reloj](./evidencias/17-reloj.png)

#### Diferencia entre multi-sitio por path y por nombre

El multi-sitio **por path** usa la misma direccion pero cambia la ruta. Por ejemplo `miweb.com/` y `miweb.com/blog`. Todo esta bajo el mismo dominio y el mismo bloque server, solo cambian los bloques location.

El multi-sitio **por nombre** (server_name) usa dominios diferentes. Por ejemplo `miweb.com` y `blog.miweb.com`. Cada dominio tiene su propio bloque server con su server_name distinto.

#### Otros tipos de multi-sitio

- **Por puerto**: El mismo servidor escucha en puertos diferentes. Por ejemplo una web en el puerto 80 y otra en el 8080.

- **Por IP**: Si el servidor tiene varias direcciones IP, cada web puede responder en una IP diferente usando la directiva `listen` con la IP concreta.

#### Mi configuracion activa

He visto el contenido del archivo dentro del contenedor con:

```bash
docker compose exec web sh -c "sed -n '1,50p' /etc/nginx/conf.d/default.conf"
```

Las directivas clave para el multi-sitio son:

- **root /usr/share/nginx/html** → Define la carpeta base donde estan los archivos
- **location /** → Bloque que maneja las peticiones a la raiz
- **location /reloj** → Bloque que maneja las peticiones a /reloj
- **try_files $uri $uri/ =404** → Busca el archivo pedido, si no existe busca como carpeta, si tampoco existe devuelve 404

Cuando alguien entra a `/reloj`, Nginx busca en `/usr/share/nginx/html/reloj/` porque combina el root con la ruta.

Evidencia: ![Configuracion dentro del contenedor](./evidencias/18-defaultconf-inside.png)

### d) Autenticacion y control de acceso

He creado una zona protegida en `/admin` que requiere usuario y contraseña para acceder.

#### Creacion del contenido protegido

Primero cree la carpeta admin con un index.html dentro del volumen compartido:

```bash
docker exec sftp-server sh -c "mkdir -p /home/usuario/upload/admin"
docker exec sftp-server sh -c "echo '<html><body><h1>Panel de Administracion</h1></body></html>' > /home/usuario/upload/admin/index.html"
```

Evidencia: ![Contenido admin](./evidencias/19-admin-html.png)

#### Configuracion de la autenticacion

Cree un archivo `.htpasswd` con el usuario y contraseña usando htpasswd:

```bash
docker run --rm httpd:alpine htpasswd -nb admin Admin1234!
```

Esto genera el hash de la contraseña que guarde en el archivo `.htpasswd`.

Luego añadi en `default.conf` un location para /admin con autenticacion basica:

```nginx
# Zona protegida con autenticacion
location /admin {
    auth_basic "Zona de Administracion";
    auth_basic_user_file /etc/nginx/.htpasswd;
    try_files $uri $uri/ =404;
}
```

- **auth_basic** → Activa la autenticacion y define el mensaje que sale en el popup
- **auth_basic_user_file** → Ruta al archivo con los usuarios y contraseñas

Evidencia: ![Configuracion auth](./evidencias/20-defaultconf-auth.png)

#### Prueba sin credenciales

```bash
curl.exe -I -k https://localhost:8443/admin/
```

Devuelve 401 Unauthorized:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Zona de Administracion"
```

Evidencia: ![Acceso sin credenciales](./evidencias/21-curl-401.png)

#### Prueba con credenciales

```bash
curl.exe -I -k -u admin:Admin1234! https://localhost:8443/admin/
```

Devuelve 200 OK:

```
HTTP/1.1 200 OK
```

Evidencia: ![Acceso con credenciales](./evidencias/22-curl-200.png)

### e) Certificados digitales
- Respuesta:
- Evidencias:
  - evidencias/e-01-ls-certs.png
  - evidencias/e-02-compose-certs.png
  - evidencias/e-03-defaultconf-ssl.png

### f) Comunicaciones seguras
- Respuesta:
- Evidencias:
  - evidencias/f-01-https.png
  - evidencias/f-02-301-network.png

### g) Documentacion
- Respuesta:
- Evidencias: enlaces a todas las capturas

### h) Ajustes para implantacion de apps
- Respuesta:
- Evidencias:
  - evidencias/h-01-root.png
  - evidencias/h-02-reloj.png

### i) Virtualizacion en despliegue
- Respuesta:
- Evidencias:
  - evidencias/i-01-compose-ps.png

### j) Logs: monitorizacion y analisis
- Respuesta:
- Evidencias:
  - evidencias/j-01-logs-follow.png
  - evidencias/j-02-metricas.png

---

## Checklist final

### Parte 1
- [ ] 1) Servicio Nginx activo
- [ ] 2) Configuracion cargada
- [ ] 3) Resolucion de nombres
- [ ] 4) Contenido Web (Cloud Academy)
- [ ] 5) Conexion SFTP exitosa
- [ ] 6) Permisos de escritura
- [ ] 7) Contenedores activos
- [ ] 8) Persistencia (Volumen compartido)
- [ ] 9) Despliegue multi-sitio (/reloj)
- [ ] 10) Cifrado SSL
- [ ] 11) Redireccion forzada (301)

### Parte 2 (RA2)
- [ ] a) Parametros de administracion
- [ ] b) Ampliacion de funcionalidad + modulo investigado
- [ ] c) Sitios virtuales / multi-sitio
- [ ] d) Autenticacion y control de acceso
- [ ] e) Certificados digitales
- [ ] f) Comunicaciones seguras
- [ ] g) Documentacion
- [ ] h) Ajustes para implantacion de apps
- [ ] i) Virtualizacion en despliegue
- [ ] j) Logs: monitorizacion y analisis
