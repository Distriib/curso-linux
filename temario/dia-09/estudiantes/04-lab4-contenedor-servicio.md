# Lab 4.4 — El contenedor como servicio

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

**Comprobar:** las dos páginas cargan sin que nadie haya iniciado sesión. Después, `active` dos veces, y `podman ps` con `systemd-web` y `miweb`.

---

# Solución — todos los comandos

```bash
# Parte 1 — limpiar los contenedores que estorban
podman rm -f web web2
podman ps -a                     # queda solo miweb

# Parte 2 — el archivo Quadlet
mkdir -p ~/.config/containers/systemd
vim ~/.config/containers/systemd/web.container
```
Contenido:
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
/usr/libexec/podman/quadlet -dryrun -user    # muestra el .service que va a generar

# Parte 3 — arrancar el servicio
systemctl --user daemon-reload
systemctl --user start web.service
systemctl --user enable web.service          # da error, y es esperado
systemctl --user status web.service --no-pager | head -6
podman ps
curl http://localhost:8080/

# Parte 4 — arrancar sin que nadie inicie sesión
loginctl show-user student | grep Linger     # Linger=no
sudo loginctl enable-linger student
loginctl show-user student | grep Linger     # Linger=yes
ls /var/lib/systemd/linger/

# Parte 5 — la forma vieja, para reconocerla
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

# Parte 6 — la prueba de verdad
sudo reboot
```
Esperar un minuto. **Antes de conectarse por SSH**, abrir en el navegador `http://192.168.56.10:8080/` y `http://192.168.56.10:8083/`. Después entrar y comprobar:
```bash
systemctl --user is-active web.service container-miweb.service
podman ps
```

**Quadlet: se escribe un `.container`, systemd genera el `.service`.** El archivo va en `~/.config/containers/systemd/` y systemd lo convierte en `web.service` cada vez que arranca. El servicio generado no existe como archivo permanente — vive en `/run/user/1000/systemd/generator/`.

**Por eso `systemctl --user enable` da error** (`Unit ... is transient or generated`) y **no importa**: el bloque `[Install] WantedBy=default.target` del `.container` ya deja el servicio habilitado. Se opera normal con `systemctl --user start|stop|restart web.service`.

**Siempre `--user`.** Estos servicios son del usuario `student`, no del sistema. Sin `--user`, systemd los busca en `/etc/systemd/system` y no los encuentra.

**El *linger* es la pieza que casi nadie recuerda.** Sin él, los servicios de usuario arrancan cuando `student` inicia sesión y **mueren cuando cierra la última**. Un servidor que se reinicia de madrugada quedaría con el contenedor caído hasta que alguien entrara por SSH. `loginctl enable-linger student` hace que systemd mantenga vivo su espacio aunque no haya ninguna sesión — y esa es la única razón por la que la Parte 6 funciona.

**Las dos formas, comparadas:**

| | Quadlet (`.container`) | `podman generate systemd` |
|---|---|---|
| Qué se escribe | un archivo corto y declarativo | se genera un `.service` largo |
| Si cambia algo | se edita el `.container` y `daemon-reload` | hay que regenerar el archivo |
| `enable` | da error (ya viene habilitado por `[Install]`) | funciona normal |
| Estado | **es lo recomendado hoy** | marcado como `DEPRECATED` |

El aviso `DEPRECATED command` que imprime la Parte 5 es correcto y esperado: está en el material para que lo reconozcas si te lo encontrás en un servidor viejo, no para que lo uses.

**El `--new` de la Parte 5** hace que el servicio **cree** el contenedor en cada arranque en vez de intentar arrancar uno existente. Por eso se puede borrar `miweb` antes y el servicio lo vuelve a crear solo.
