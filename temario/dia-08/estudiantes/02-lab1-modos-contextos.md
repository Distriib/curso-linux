# Lab 2.1 — Modos y contextos

Vamos a confirmar que el Apache caído del bloque anterior es culpa de SELinux, volver a dejarlo sano, y leer etiquetas de archivos, procesos y de tu sesión.

---

## Parte 1 — En qué modo estamos

**¿SELinux está prendido?**

```bash
getenforce
sestatus
```

**Comprobar:**
```
Enforcing
SELinux status:                 enabled
...
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
...
```

---

## Parte 2 — La prueba del permissive

**Si SELinux deja de aplicar la política, ¿Apache arranca en el 82?**

```bash
sudo setenforce 0
getenforce
sudo systemctl restart httpd
curl -I http://localhost:82
sudo ausearch -m AVC -ts recent | tail -1
```

**Comprobar:**
```
Permissive
HTTP/1.1 200 OK
...
type=AVC msg=audit(...): avc:  denied  { name_bind } for  pid=... comm="httpd" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=1
```
Arrancó: **era SELinux**. Y la denegación quedó registrada igual (`permissive=1`). Eso es lo que se hace en producción para diagnosticar: unos segundos, no días.

---

## Parte 3 — Volver a enforcing y dejar Apache sano

**¿Cómo dejo Apache funcionando sin apagar SELinux?**

```bash
sudo setenforce 1
sudo systemctl restart httpd
sudo rm /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
systemctl is-active httpd
grep -v "#" /etc/selinux/config
```

**Comprobar:**
```
Job for httpd.service failed ...
active
SELINUX=enforcing
SELINUXTYPE=targeted
```
En enforcing, el 82 vuelve a fallar. Sin el `Listen 82`, Apache arranca en el 80. El archivo de arranque dice `enforcing`: no lo tocamos. El 82 lo vamos a arreglar **bien** en el Bloque 3.

---

## Parte 4 — Leer etiquetas

**¿Qué etiqueta tiene mi sesión, Apache y los archivos?**

```bash
id -Z
ps -eZ | grep httpd
ls -Z /var/www/html/
ls -Zd /root /home/student /etc/shadow /var/www/html /tmp
```

**Comprobar:**
```
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
system_u:system_r:httpd_t:s0        ...  httpd
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
system_u:object_r:admin_home_t:s0 /root
unconfined_u:object_r:user_home_dir_t:s0 /home/student
system_u:object_r:shadow_t:s0 /etc/shadow
system_u:object_r:httpd_sys_content_t:s0 /var/www/html
system_u:object_r:tmp_t:s0 /tmp
```
El primer campo (`system_u` / `unconfined_u`) varía según quién creó el archivo; lo que decide es el **tipo**. Tu shell es `unconfined_t`: SELinux no te limita a vos, limita a los servicios.

---

# Solución — todos los comandos

```bash
# Parte 1 — en qué modo estamos
getenforce
sestatus

# Parte 2 — la prueba del permissive
sudo setenforce 0
getenforce
sudo systemctl restart httpd     # ahora SÍ arranca en el 82
curl -I http://localhost:82
sudo ausearch -m AVC -ts recent | tail -1

# Parte 3 — volver a enforcing y dejar Apache sano
sudo setenforce 1
sudo systemctl restart httpd     # vuelve a fallar: era SELinux
sudo rm /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd     # sin el Listen 82, arranca en el 80
systemctl is-active httpd
grep -v "#" /etc/selinux/config

# Parte 4 — leer etiquetas
id -Z
ps -eZ | grep httpd
ls -Z /var/www/html/
ls -Zd /root /home/student /etc/shadow /var/www/html /tmp
```

**La prueba del permissive** es para **diagnosticar**, no para arreglar: se pone permissive unos segundos, se confirma que era SELinux, y se vuelve a `enforcing` **antes** de corregir. La denegación queda registrada igual (`permissive=1` en el AVC), que es justamente lo que sirve.

El puerto 82 se arregla **bien** en el Lab 3.1, con `semanage`. Dejar `setenforce 0` sería tapar el problema.
