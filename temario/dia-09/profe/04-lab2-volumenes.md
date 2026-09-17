# Lab — Volúmenes y SELinux (comandos)

## Parte 1 — Sin `:Z`
```bash
mkdir -p ~/web
echo "<h1>Hola desde el volumen ~/web en rhel01</h1>" > ~/web/index.html
ls -Z ~/web/index.html
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html registry.access.redhat.com/ubi9/httpd-24
curl -sI http://localhost:8081/ | head -1
```
Decir: "permisos Linux perfectos (`644`) y 403. Este sí es de permisos."

## Parte 2 — Leer quién dijo que no
```bash
podman logs web2 | tail -1
sudo ausearch -m AVC -ts recent | grep user_home_t
```
Decir: "`container_t` quiso leer `user_home_t`. SELinux protege tu home del contenedor, y hoy eso nos molesta. Es el mismo `ausearch` del Día 8."

El permiso denegado puede ser `read`, `getattr` o `search`: lo que importa es `tcontext=...user_home_t`.

## Parte 3 — Con `:Z`
```bash
podman rm -f web2
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html:Z registry.access.redhat.com/ubi9/httpd-24
curl http://localhost:8081/
ls -Z ~/web/index.html
```
Decir: "la etiqueta cambió en el servidor: `container_file_t` con dos categorías que son de este contenedor. Eso es `:Z`."

## Parte 4 — Es la misma carpeta, no una copia
```bash
echo "<p>Actualizado desde el servidor</p>" >> ~/web/index.html
curl http://localhost:8081/
```

## Parte 5 — Abrir los puertos para el navegador
```bash
sudo firewall-cmd --permanent --add-port=8080/tcp --add-port=8081/tcp --add-port=8083/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=8080/tcp --add-port=8081/tcp --add-port=8083/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
```
Ellos, en el navegador: `http://192.168.56.10:8081/`. Vos: `http://<tu IP host-only>:8081/`.

Decir: "no hizo falta `semanage port` como con Apache el Día 8: acá quien abre el puerto es un proceso de `student`, no un servicio confinado."

Si en alguna zona aparecen también `82/tcp` u `8082/tcp` del Día 8: no molestan. Quien no tenga red host-only: agregar en VirtualBox la regla NAT `8081 → 8081` (se puede con la VM encendida) y abrir `http://localhost:8081/`.
