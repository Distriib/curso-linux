# Comandos — Podman: contenedores sin Docker

## Vocabulario

| Palabra | Qué es |
|---|---|
| **Imagen** | la plantilla, de solo lectura, armada en capas: sistema base + programa + configuración |
| **Contenedor** | una imagen **en ejecución**: un proceso normal de Linux, aislado, con su propia capa de escritura que desaparece al borrarlo |
| **Registro** | el catálogo de donde se descargan las imágenes |
| **UBI** | *Universal Base Image*: RHEL para contenedores, sin suscripción |
| **Volumen** | una carpeta del servidor que el contenedor ve adentro; lo que tiene que persistir va ahí |
| **rootless** | cada usuario tiene sus propios contenedores, sin ser root |

## El nombre completo de una imagen

```
registry.access.redhat.com/ubi9/httpd-24:latest
        │                    │      │       └ etiqueta (versión); si falta, latest
        │                    │      └ nombre
        │                    └ espacio (quién la publica)
        └ registro
```

Siempre el nombre completo. `podman pull httpd` a secas pregunta en qué registro buscar.

| Registro | Qué tiene |
|---|---|
| `registry.access.redhat.com` | imágenes UBI, sin usuario |
| `registry.redhat.io` | el catálogo completo de Red Hat, pide `podman login` |
| `docker.io` | Docker Hub |

## Podman y Docker

Misma línea de comandos. La diferencia: Podman **no tiene un servicio central como root**. Cada contenedor es un proceso del usuario que lo lanzó. Los de `student` viven en `~/.local/share/containers/`; los de root en `/var/lib/containers/` y **no se ven** entre sí.

Un usuario sin privilegios **no puede publicar puertos menores de 1024** en el servidor. Adentro del contenedor sí puede escuchar en el 80: el límite es el puerto **del servidor**.

## Imágenes

| Comando | Qué hace |
|---|---|
| `podman search registry.access.redhat.com/ubi9 --limit 5` | buscar |
| `podman pull registry.access.redhat.com/ubi9/ubi` | descargar |
| `podman images` | las que tengo |
| `podman image inspect imagen` | qué trae adentro: usuario (`User`), puertos (`ExposedPorts`) |
| `podman rmi imagen` | borrar |
| `podman tag miweb miweb:1.0` | otro nombre para la misma imagen |
| `podman image prune -f` | borrar las que quedaron sin nombre |
| `skopeo inspect docker://registry.access.redhat.com/ubi9/ubi` | ver una imagen del registro sin descargarla |

## Ejecutar

```
podman run -d --name web -p 8080:8080 -v ~/web:/var/www/html:Z registry.access.redhat.com/ubi9/httpd-24
            │     │          │              │                 │                 └ imagen
            │     │          │              │                 └ :Z = etiqueta SELinux para este contenedor
            │     │          │              └ -v servidor:contenedor → volumen
            │     │          └ -p servidor:contenedor → puerto
            │     └ nombre del contenedor
            └ -d = en segundo plano
```

| Opción | Qué hace |
|---|---|
| `-it ... bash` | interactivo: entrar con una terminal |
| `--rm` | borrar el contenedor al salir |
| `-d` | dejarlo corriendo en segundo plano |

## El día a día

| Comando | Qué hace |
|---|---|
| `podman ps` / `podman ps -a` | los que corren / todos, incluidos los detenidos |
| `podman logs web` | lo que el programa imprimió: el `journalctl` del contenedor |
| `podman exec -it web bash` | abrir una terminal adentro de uno que ya corre |
| `podman top web` | sus procesos |
| `podman port web` | sus puertos |
| `podman inspect web` | todo sobre él |
| `podman stats --no-stream web` | CPU y memoria, una vez |
| `podman stop web` / `start web` / `rm -f web` | detener / arrancar / borrar aunque esté corriendo |

## SELinux y volúmenes

Una carpeta del servidor tiene su etiqueta (`user_home_t` en tu home). El contenedor corre como `container_t`, que **no puede leerla**. `:Z` la reetiqueta a `container_file_t` con una marca privada de ese contenedor.

| Sufijo | Cuándo |
|---|---|
| `:Z` | la carpeta es de **un** contenedor |
| `:z` | la comparten **varios** contenedores |

Nunca `:Z` sobre `/home` completo ni sobre `/etc`.

## Imagen propia: `Containerfile`

```
FROM registry.access.redhat.com/ubi9/ubi                   ← de qué imagen parto
RUN dnf -y install httpd && dnf clean all                  ← qué instalo (una capa)
RUN mkdir -p /run/httpd                                     ← carpeta que httpd necesita al arrancar
RUN echo "<h1>Imagen miweb construida en rhel01</h1>" > /var/www/html/index.html
EXPOSE 80                                                   ← en qué puerto escucha adentro
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]                     ← qué corre al arrancar
```

`podman build -t miweb .` la construye (el `.` = la carpeta actual). Queda como `localhost/miweb`.

## El contenedor como servicio: Quadlet

`podman run -d` muere con la sesión y no vuelve al reiniciar. **Quadlet**: un archivo `nombre.container` en `~/.config/containers/systemd/` que systemd convierte en `nombre.service`.

```
[Unit]
Description=Servidor web intranet en contenedor (Quadlet)

[Container]
Image=registry.access.redhat.com/ubi9/httpd-24     ← la imagen
PublishPort=8080:8080                              ← el -p
Volume=/home/student/web:/var/www/html:Z           ← el -v (ruta completa, sin ~)
ContainerName=web                                  ← sin esta línea se llama systemd-web

[Service]
Restart=always                                     ← si muere, vuelve

[Install]
WantedBy=default.target                            ← arranca con la sesión del usuario
```

| Comando | Qué hace |
|---|---|
| `/usr/libexec/podman/quadlet -dryrun -user` | mostrar el servicio que se va a generar (y los errores) |
| `systemctl --user daemon-reload` | que systemd lea el archivo nuevo |
| `systemctl --user start web.service` | arrancar |
| `systemctl --user status web.service` | estado; `enable` **no se usa**: el `[Install]` ya lo deja habilitado |
| `journalctl --user -u web.service` | sus logs |
| `sudo loginctl enable-linger student` | que los servicios de `student` arranquen **sin que nadie inicie sesión** |
| `podman generate systemd --new --files --name miweb` | la forma vieja: genera un `.service` a partir de un contenedor que ya corre |
