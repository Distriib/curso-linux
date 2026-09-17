# Lab — Primeros contenedores

Vamos a instalar Podman, descargar dos imágenes UBI y operar un servidor web en contenedor con los comandos de todos los días.

| Imagen | Para qué |
|---|---|
| `registry.access.redhat.com/ubi9/ubi` | RHEL 9 pelado, para entrar y mirar |
| `registry.access.redhat.com/ubi9/httpd-24` | Apache listo, escucha en el 8080 |

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

Todos a la vez. Foto.

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

Todos a la vez. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

**Comprobar:** después del `stop`, `podman ps` no lo muestra y `podman ps -a` lo muestra como `Exited (0) ... ago`. Después del `start`, `Up ... seconds`.

---

## Parte 7 — El límite del puerto 80

**¿Por qué un usuario no puede publicar en el puerto 80?**

```bash
podman run -d --name web80 -p 80:8080 registry.access.redhat.com/ubi9/httpd-24
podman rm -f web80
sudo sysctl net.ipv4.ip_unprivileged_port_start
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** el primer comando falla con un error al reservar el puerto 80 (`permission denied` o `address already in use`). El último dice `net.ipv4.ip_unprivileged_port_start = 1024`: por debajo de ese puerto, solo root. Además, el 80 ya lo usa el Apache del servidor. La práctica: publicar arriba de 1024.
