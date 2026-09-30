# Lab 4.1 — Primeros contenedores

Vamos a instalar Podman, descargar dos imágenes UBI y operar un servidor web en contenedor con los comandos de todos los días.

| Imagen | Para qué |
|---|---|
| `registry.access.redhat.com/ubi9/ubi` | RHEL 9 pelado, para entrar y mirar |
| `registry.access.redhat.com/ubi9/httpd-24` | Apache listo, escucha en el 8080 |

> **Antes de empezar, comprobar el espacio libre:** las imágenes de este bloque ocupan cerca de 1 GB.
> ```bash
> df -h /
> ```
> Tiene que haber al menos **3 GB** libres. Si no alcanzan: `podman image prune -f` y `sudo dnf clean all`.

---

## Parte 1 — Instalar y comprobar que somos rootless

**¿Dónde guarda `student` sus contenedores, y por qué no hace falta ser root?**

```bash
sudo dnf install -y container-tools
podman --version
podman info | grep rootless
podman info | grep graphRoot
grep student /etc/subuid /etc/subgid
```

**Comprobar:**
```
podman version 5.x
    rootless: true
  graphRoot: /home/student/.local/share/containers/storage
  ...
/etc/subuid:student:100000:65536
/etc/subgid:student:100000:65536
```
`student` tiene 65536 números de usuario reservados: con eso el contenedor puede tener "su root" sin ser root.

---

## Parte 2 — Registros y descarga

**¿En qué registros busca Podman cuando no le doy el nombre completo?**

```bash
grep unqualified /etc/containers/registries.conf
podman search registry.access.redhat.com/ubi9 --limit 5
podman pull registry.access.redhat.com/ubi9/ubi
podman pull registry.access.redhat.com/ubi9/httpd-24
```

**Comprobar:**
```bash
podman images
```
```
unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "docker.io"]
NAME                                        DESCRIPTION
registry.access.redhat.com/ubi9/ubi         Provides the latest release of the Red Hat Universal Base Image 9.
...
Trying to pull registry.access.redhat.com/ubi9/ubi:latest...
Getting image source signatures
Copying blob ...
Writing manifest to image destination
...
REPOSITORY                                TAG     IMAGE ID      CREATED      SIZE
registry.access.redhat.com/ubi9/httpd-24  latest  ...           ...          ... MB
registry.access.redhat.com/ubi9/ubi       latest  ...           ...          ... MB
```

---

## Parte 3 — Entrar a un contenedor

**Adentro soy root. ¿Quién soy afuera?**

```bash
podman run -it --rm registry.access.redhat.com/ubi9/ubi bash
```
Adentro (el prompt cambia a `[root@... /]#`):
```
head -2 /etc/os-release
ps aux
id
exit
```
```bash
podman ps -a
```

**Comprobar:**
```
NAME="Red Hat Enterprise Linux"
VERSION="9.x (Plow)"
USER  PID ... COMMAND
root    1 ... bash
root    8 ... ps aux
uid=0(root) gid=0(root) groups=0(root)
```
Dos procesos en total: `bash` es el PID 1. No hay systemd ni nada más. `podman ps -a` no lo muestra: `--rm` lo borró al salir.

---

## Parte 4 — Mirar la imagen web y lanzarla

**¿En qué puerto escucha esta imagen y como qué usuario corre?**

```bash
podman image inspect registry.access.redhat.com/ubi9/httpd-24 | grep -A 3 ExposedPorts
podman image inspect registry.access.redhat.com/ubi9/httpd-24 | grep User
sudo ss -tlnp | grep 8080
podman run -d --name web -p 8080:8080 registry.access.redhat.com/ubi9/httpd-24
podman ps
curl -sI http://localhost:8080/ | head -3
```

**Comprobar:**
```
        "ExposedPorts": {
            "8080/tcp": {},
            "8443/tcp": {}
        },
        "User": "1001",
CONTAINER ID  IMAGE                                            COMMAND     CREATED        STATUS        PORTS                   NAMES
...           registry.access.redhat.com/ubi9/httpd-24:latest  ...         5 seconds ago  Up 5 seconds  0.0.0.0:8080->8080/tcp  web
HTTP/1.1 403 Forbidden
Date: ...
Server: Apache/2.4.x (Red Hat Enterprise Linux)
```
El `ss` no devuelve nada: el 8080 estaba libre. El `403` es la página de prueba de Apache: todavía no hay contenido nuestro.

---

## Parte 5 — Los comandos de todos los días

**¿Cómo veo qué pasa adentro sin entrar? ¿Y entrando?**

```bash
podman logs web | tail -3
podman port web
podman top web
podman exec -it web bash
```
Adentro:
```
id
ls -ld /var/www/html
httpd -v
exit
```
```bash
podman inspect web | grep Status
podman stats --no-stream web
```

**Comprobar:**
```
[...] AH00489: Apache/2.4.x (Red Hat Enterprise Linux) ... configured -- resuming normal operations
8080/tcp -> 0.0.0.0:8080
USER  PID  PPID  ...  COMMAND
1001  1    0     ...  httpd -D FOREGROUND
uid=1001(default) gid=0(root) groups=0(root)
drwxrwxr-x. 2 default root 6 ... /var/www/html
Server version: Apache/2.4.x (Red Hat Enterprise Linux)
            "Status": "running",
ID  NAME  CPU %  MEM USAGE / LIMIT  ...
... web   0.00%  ... MB / ... GB     ...
```
`podman logs` es la salida del programa principal: el `journalctl` de los contenedores.

---

## Parte 6 — Detener y arrancar

**¿Qué diferencia hay entre `podman ps` y `podman ps -a`?**

```bash
podman stop web
podman ps
podman ps -a
podman start web
podman ps
```

**Comprobar:** después del `stop`, `podman ps` no lo muestra y `podman ps -a` lo muestra como `Exited (0) ... ago`. Después del `start`, `Up ... seconds`.

---

## Parte 7 — El límite del puerto 80

**¿Por qué un usuario no puede publicar en el puerto 80?**

```bash
podman run -d --name web80 -p 80:8080 registry.access.redhat.com/ubi9/httpd-24
podman rm -f web80
sudo sysctl net.ipv4.ip_unprivileged_port_start
```

**Comprobar:** el primer comando falla con un error al reservar el puerto 80 (`permission denied` o `address already in use`). El último dice `net.ipv4.ip_unprivileged_port_start = 1024`: por debajo de ese puerto, solo root. Además, el 80 ya lo usa el Apache del servidor. La práctica: publicar arriba de 1024.

---

# Solución — todos los comandos

```bash
# Parte 1 — instalar y comprobar que somos rootless
sudo dnf install -y container-tools
podman --version
podman info | grep rootless
podman info | grep graphRoot
grep student /etc/subuid /etc/subgid

# Parte 2 — registros y descarga
grep unqualified /etc/containers/registries.conf
podman search registry.access.redhat.com/ubi9 --limit 5
podman pull registry.access.redhat.com/ubi9/ubi
podman pull registry.access.redhat.com/ubi9/httpd-24
podman images

# Parte 3 — entrar a un contenedor
podman run -it --rm registry.access.redhat.com/ubi9/ubi bash
```
Adentro (el prompt cambia a `[root@... /]#`):
```
head -2 /etc/os-release
ps aux
id
exit
```
```bash
podman ps -a          # no aparece: --rm lo borró al salir

# Parte 4 — mirar la imagen web y lanzarla
podman image inspect registry.access.redhat.com/ubi9/httpd-24 | grep -A 3 ExposedPorts
podman image inspect registry.access.redhat.com/ubi9/httpd-24 | grep User
sudo ss -tlnp | grep 8080
podman run -d --name web -p 8080:8080 registry.access.redhat.com/ubi9/httpd-24
podman ps
curl -sI http://localhost:8080/ | head -3

# Parte 5 — los comandos de todos los días
podman logs web | tail -3
podman port web
podman top web
podman exec -it web bash
```
Adentro:
```
id
ls -ld /var/www/html
httpd -v
exit
```
```bash
podman inspect web | grep Status
podman stats --no-stream web

# Parte 6 — detener y arrancar
podman stop web
podman ps               # no lo muestra
podman ps -a            # Exited (0)
podman start web
podman ps               # Up ... seconds

# Parte 7 — el límite del puerto 80
podman run -d --name web80 -p 80:8080 registry.access.redhat.com/ubi9/httpd-24   # falla
podman rm -f web80
sudo sysctl net.ipv4.ip_unprivileged_port_start
```

**Rootless: el contenedor cree que es root, pero afuera es `student`.** Las líneas de `/etc/subuid` y `/etc/subgid` le reservan a `student` 65536 números de usuario. El `root` de adentro (UID 0) se traduce afuera al UID 100000. Si alguien se escapa del contenedor, sale como un usuario sin ningún privilegio — no como root de la máquina.

Por eso todo se guarda en `~/.local/share/containers/storage` y no en `/var/lib/containers`, y por eso ningún `podman` de este lab lleva `sudo`.

**Anatomía de `podman run`:**
```
podman run -d --name web -p 8080:8080 imagen
             │      │        │    └ puerto ADENTRO del contenedor
             │      │        └ puerto del SERVIDOR (el que se abre en el firewall)
             │      └ nombre, para no tener que usar el ID
             └ -d = en segundo plano · -it = interactivo · --rm = borrarlo al salir
```

**El `403` de la Parte 4 no es un error:** es la página de bienvenida de Apache, porque todavía no hay contenido nuestro. Eso llega en el Lab 4.2 con el volumen.

**Los comandos de diagnóstico, comparados con lo que ya sabés:**

| Contenedor | Equivalente en el servidor |
|---|---|
| `podman logs web` | `journalctl -u httpd` |
| `podman top web` | `ps aux` |
| `podman exec -it web bash` | entrar por SSH |
| `podman stats` | `top` |

**Por qué falla el puerto 80.** Dos razones a la vez: un proceso sin privilegios no puede reservar puertos por debajo de 1024 (`ip_unprivileged_port_start = 1024`), y además el 80 ya lo ocupa el Apache del servidor. La práctica normal con contenedores rootless es publicar siempre arriba de 1024.
