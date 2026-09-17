# Lab — Volúmenes y SELinux

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
success
success
success
8080/tcp 8081/tcp 8083/tcp
8080/tcp 8081/tcp 8083/tcp
```
El navegador muestra "Hola desde el volumen". No hizo falta `semanage port`: quien abre el puerto es un proceso de `student`, no un servicio confinado. El 8083 se abre ahora porque lo usa la imagen del próximo lab.
