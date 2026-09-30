# Lab 4.2 — Volúmenes y SELinux

Vamos a hacer que el contenedor sirva una carpeta del servidor, ver cómo SELinux lo bloquea, corregirlo con `:Z`, y abrir los puertos para el navegador.

| Qué | Valor |
|---|---|
| Carpeta del servidor | `~/web` |
| Adentro del contenedor | `/var/www/html` |
| Contenedor | `web2`, puerto `8081` |

---

## Parte 1 — Sin `:Z`

**Los permisos Linux están bien. ¿Por qué da 403?**

```bash
mkdir -p ~/web
echo "<h1>Hola desde el volumen ~/web en rhel01</h1>" > ~/web/index.html
ls -Z ~/web/index.html
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html registry.access.redhat.com/ubi9/httpd-24
curl -sI http://localhost:8081/ | head -1
```

**Comprobar:**
```
unconfined_u:object_r:user_home_t:s0 /home/student/web/index.html
HTTP/1.1 403 Forbidden
```

---

## Parte 2 — Leer quién dijo que no

**¿Qué etiqueta quería leer el contenedor y cuál tenía el archivo?**

```bash
podman logs web2 | tail -1
sudo ausearch -m AVC -ts recent | grep user_home_t
```

**Comprobar:**
```
[...] AH00035: access to /index.html denied (filesystem path '/var/www/html/index.html') because search permissions are missing on a component of the path
type=AVC msg=audit(...): avc:  denied  { read } for  pid=... comm="httpd" name="index.html" ... scontext=system_u:system_r:container_t:s0:c...,c... tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
```
`container_t` intentó leer `user_home_t`. Ese 403 sí es de permisos: de SELinux.

---

## Parte 3 — Con `:Z`

**¿Qué cambió en el archivo del servidor?**

```bash
podman rm -f web2
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html:Z registry.access.redhat.com/ubi9/httpd-24
curl http://localhost:8081/
ls -Z ~/web/index.html
```

**Comprobar:**
```
<h1>Hola desde el volumen ~/web en rhel01</h1>
unconfined_u:object_r:container_file_t:s0:c...,c... /home/student/web/index.html
```
La etiqueta cambió **en el servidor**: `container_file_t`, con dos categorías que son de este contenedor.

---

## Parte 4 — Es la misma carpeta, no una copia

**Si edito el archivo en el servidor, ¿el contenedor lo ve?**

```bash
echo "<p>Actualizado desde el servidor</p>" >> ~/web/index.html
curl http://localhost:8081/
```

**Comprobar:**
```
<h1>Hola desde el volumen ~/web en rhel01</h1>
<p>Actualizado desde el servidor</p>
```
Al instante, sin reiniciar nada.

---

## Parte 5 — Abrir los puertos para el navegador

**¿Hace falta `semanage port` como con Apache?**

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp --add-port=8081/tcp --add-port=8083/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=8080/tcp --add-port=8081/tcp --add-port=8083/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
```
En el navegador de tu computadora: `http://192.168.56.10:8081/`.

**Comprobar:**
```
success
success
success
8080/tcp 8081/tcp 8083/tcp
8080/tcp 8081/tcp 8083/tcp
```
El navegador muestra "Hola desde el volumen". No hizo falta `semanage port`: quien abre el puerto es un proceso de `student`, no un servicio confinado. El 8083 se abre ahora porque lo usa la imagen del próximo lab.

---

# Solución — todos los comandos

```bash
# Parte 1 — sin :Z (a propósito)
mkdir -p ~/web
echo "<h1>Hola desde el volumen ~/web en rhel01</h1>" > ~/web/index.html
ls -Z ~/web/index.html                      # user_home_t
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html registry.access.redhat.com/ubi9/httpd-24
curl -sI http://localhost:8081/ | head -1   # 403 Forbidden

# Parte 2 — leer quién dijo que no
podman logs web2 | tail -1
sudo ausearch -m AVC -ts recent | grep user_home_t

# Parte 3 — con :Z
podman rm -f web2
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html:Z registry.access.redhat.com/ubi9/httpd-24
curl http://localhost:8081/
ls -Z ~/web/index.html                      # ahora container_file_t

# Parte 4 — es la misma carpeta, no una copia
echo "<p>Actualizado desde el servidor</p>" >> ~/web/index.html
curl http://localhost:8081/

# Parte 5 — abrir los puertos en las DOS zonas
sudo firewall-cmd --permanent --add-port=8080/tcp --add-port=8081/tcp --add-port=8083/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=8080/tcp --add-port=8081/tcp --add-port=8083/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
# y en el navegador de tu computadora: http://192.168.56.10:8081/
```

**Qué hace `-v origen:destino`.** Monta una carpeta del servidor adentro del contenedor. **No es una copia:** es la misma carpeta vista desde los dos lados, por eso el `echo` de la Parte 4 se ve al instante sin reiniciar nada. Así se separa el contenido (que es tuyo y sobrevive) de la imagen (que es descartable).

**El `403` de la Parte 1 sí era de permisos — de SELinux.** El proceso de adentro corre como `container_t` y solo puede leer `container_file_t`; el archivo tenía `user_home_t`, que es lo que le toca a cualquier cosa creada en tu `home`.

| Sufijo | Qué hace |
|---|---|
| `:Z` (mayúscula) | reetiqueta la carpeta como `container_file_t` **privada de este contenedor** (le pone dos categorías propias) |
| `:z` (minúscula) | la reetiqueta como **compartida** entre varios contenedores |
| sin sufijo | no toca la etiqueta → 403 |

**Cuidado con dos `:Z` sobre la misma carpeta:** el segundo contenedor le pone sus propias categorías y deja al primero afuera. Si dos contenedores tienen que compartir la carpeta, va `:z` minúscula.

**La etiqueta cambia en el servidor, no adentro.** Después del `:Z`, `ls -Z ~/web/index.html` en la VM ya muestra `container_file_t`. Es un cambio real y permanente sobre tus archivos.

**Acá no hizo falta `semanage port`** como sí hizo falta con Apache el Día 8: el que abre el 8081 es un proceso de `student` sin confinar, no un servicio bajo política. El firewall sí hace falta igual, y en las dos zonas.
