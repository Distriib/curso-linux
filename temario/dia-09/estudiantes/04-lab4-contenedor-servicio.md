# Lab — El contenedor como servicio

Vamos a hacer que el servidor web en contenedor arranque con el sistema, sin que nadie inicie sesión, manejado por systemd.

| Qué | Valor |
|---|---|
| Archivo Quadlet | `~/.config/containers/systemd/web.container` |
| Servicio que genera | `web.service` (del usuario), contenedor `systemd-web`, puerto `8080` |
| Forma vieja | `container-miweb.service`, generado desde el contenedor `miweb` |

---

## Parte 1 — Limpiar

**¿Por qué hay que borrar `web` y `web2` antes?**

```bash
podman rm -f web web2
podman ps -a
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** queda solo `miweb`, `Up ...`. `web` usaba el 8080 que va a usar el servicio, y `web2` tenía `:Z` sobre la misma carpeta `~/web`: dos `:Z` sobre la misma carpeta se pisan la etiqueta.

---

## Parte 2 — El archivo Quadlet

**¿Qué servicio va a generar systemd a partir de esto?**

```bash
mkdir -p ~/.config/containers/systemd
vim ~/.config/containers/systemd/web.container
```
Pegar:
```
[Unit]
Description=Servidor web intranet en contenedor (Quadlet)

[Container]
Image=registry.access.redhat.com/ubi9/httpd-24
PublishPort=8080:8080
Volume=/home/student/web:/var/www/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```
```bash
/usr/libexec/podman/quadlet -dryrun -user
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** imprime el servicio completo que va a generar. Importan estas líneas:
```
SourcePath=/home/student/.config/containers/systemd/web.container
Restart=always
WantedBy=default.target
ExecStart=/usr/bin/podman run --name systemd-web ... -v /home/student/web:/var/www/html:Z --publish 8080:8080 registry.access.redhat.com/ubi9/httpd-24
```
El contenedor se va a llamar `systemd-web`. Si el archivo tiene un error, acá lo dice.

---

## Parte 3 — Arrancar el servicio

**¿Por qué `enable` da error y no importa?**

```bash
systemctl --user daemon-reload
systemctl --user start web.service
systemctl --user enable web.service
systemctl --user status web.service --no-pager | head -6
podman ps
curl http://localhost:8080/
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Failed to enable unit: Unit /run/user/1000/systemd/generator/web.service is transient or generated.
● web.service - Servidor web intranet en contenedor (Quadlet)
     Loaded: loaded (/home/student/.config/containers/systemd/web.container; generated)
     Active: active (running) since ...
   Main PID: ... (conmon)
CONTAINER ID  IMAGE  ...  PORTS                   NAMES
...                  ...  0.0.0.0:8080->8080/tcp  systemd-web
...                  ...  0.0.0.0:8083->80/tcp    miweb
<h1>Hola desde el volumen ~/web en rhel01</h1>
<p>Actualizado desde el servidor</p>
```
El error de `enable` es esperado: el servicio es generado y el `[Install]` ya lo deja habilitado. Desde ahora se maneja con `systemctl --user stop|start|restart web.service`.

---

## Parte 4 — Arrancar sin que nadie inicie sesión

**Si nadie entra por SSH después del reinicio, ¿quién arranca el servicio?**

```bash
loginctl show-user student | grep Linger
sudo loginctl enable-linger student
loginctl show-user student | grep Linger
ls /var/lib/systemd/linger/
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Linger=no
Linger=yes
student
```
Sin *linger*, los servicios de `student` mueren al cerrar la última sesión y no arrancan al reiniciar.

---

## Parte 5 — La forma vieja

**¿Qué diferencia hay entre generar el `.service` y escribir el `.container`?**

```bash
cd ~
podman generate systemd --new --files --name miweb
mkdir -p ~/.config/systemd/user
mv container-miweb.service ~/.config/systemd/user/
grep ExecStart ~/.config/systemd/user/container-miweb.service
podman rm -f miweb
systemctl --user daemon-reload
systemctl --user enable --now container-miweb.service
systemctl --user is-active container-miweb.service
podman ps
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
DEPRECATED command:
It is recommended to use Quadlets for running containers and pods under systemd.
...
/home/student/container-miweb.service
ExecStart=/usr/bin/podman run --cidfile=%t/%n.ctr-id --cgroups=no-conmon --rm --sdnotify=conmon --replace -d --name miweb -p 8083:80 miweb
Created symlink /home/student/.config/systemd/user/default.target.wants/container-miweb.service → ...
active
```
`podman ps` muestra `systemd-web` y `miweb` arriba. Acá `enable --now` sí funciona: es un `.service` normal. `--new` hace que el servicio **cree** el contenedor en cada arranque; por eso pudimos borrarlo antes.

---

## Parte 6 — La prueba de verdad

**¿Están arriba antes de que alguien entre?**

```bash
sudo reboot
```
Esperar un minuto. **Antes de conectarse por SSH**, en el navegador de tu computadora: `http://192.168.56.10:8080/` y `http://192.168.56.10:8083/`. Después entrar por SSH:
```bash
systemctl --user is-active web.service container-miweb.service
podman ps
```

Todos a la vez. Foto.

**Comprobar:** las dos páginas cargan sin que nadie haya iniciado sesión. Después, `active` dos veces, y `podman ps` con `systemd-web` y `miweb`.
