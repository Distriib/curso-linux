# Comandos — Podman (guía del instructor)

Diez minutos, sin tipear: Podman todavía no está instalado. Es lectura de tablas y diagramas. Cuatro cosas que tienen que quedar: (1) imagen = plantilla, contenedor = la plantilla corriendo; (2) Podman no tiene servicio central y cada usuario tiene lo suyo (rootless); (3) una carpeta del servidor dentro del contenedor lleva `:Z` o SELinux la bloquea; (4) para que arranque con el sistema: Quadlet + `loginctl enable-linger`. Los cuatro se practican en los cuatro labs.

**Qué es un contenedor:** un proceso normal de Linux que corre aislado: ve su propio árbol de archivos, sus propios procesos y su propia red. No es una máquina virtual: no tiene kernel propio ni arranca un sistema; arranca **un programa**. Por eso pesa poco y arranca en un segundo.
**Qué es una imagen:** la plantilla de la que sale el contenedor. De solo lectura, armada en **capas** (sistema base, programa instalado, configuración). De una imagen salen tantos contenedores como se quiera.
**Qué es la capa de escritura:** lo que el contenedor modifica mientras corre se guarda en una capa propia, encima de la imagen. Al borrar el contenedor, esa capa desaparece. Por eso lo que tiene que durar va en un **volumen**.
**Qué es un volumen:** una carpeta del servidor que el contenedor ve adentro en una ruta que elegimos (`-v ~/web:/var/www/html`). Lo que se escribe ahí queda en el servidor.
**Qué es un registro:** el sitio de donde se descargan las imágenes. `registry.access.redhat.com` tiene las imágenes UBI y no pide usuario. `registry.redhat.io` es el catálogo completo de Red Hat y pide `podman login`. `docker.io` es Docker Hub.
**Qué es UBI:** *Universal Base Image*: RHEL empaquetado para contenedores, que Red Hat deja usar y redistribuir sin suscripción. `ubi9/ubi` es RHEL 9 pelado; `ubi9/httpd-24` es RHEL 9 con Apache listo.
**Qué es una etiqueta (tag):** la versión de la imagen, después de los dos puntos: `httpd-24:latest`. Sin etiqueta, Podman pone `latest`.
**Qué es un nombre corto:** `httpd` en vez de `registry.access.redhat.com/ubi9/httpd-24`. RHEL viene configurado para no adivinar: pregunta en qué registro buscar. Regla del curso: siempre el nombre completo.
**Qué es Podman:** la herramienta de RHEL para contenedores. Misma línea de comandos que Docker. `container-tools` es el paquete que lo instala junto con `skopeo` y `buildah`.
**Qué es Docker:** la herramienta que popularizó los contenedores. Corre un servicio central como root del que dependen todos los contenedores. Podman hace lo mismo sin ese servicio.
**Qué es rootless:** correr contenedores como un usuario común. Los contenedores de `student` son procesos de `student` y viven en `~/.local/share/containers/`. Los de root, en `/var/lib/containers/`, y ninguno ve los del otro. Un contenedor comprometido en rootless es un proceso de `student`, no de root.
**Qué es `/etc/subuid` / `/etc/subgid`:** los rangos de números de usuario que cada usuario tiene reservados para sus contenedores. `student:100000:65536` = del 100000 al 165535. Con eso, el "root" de adentro del contenedor es en realidad un número de ese rango afuera.
**Qué es `ip_unprivileged_port_start`:** un parámetro del kernel (`sysctl`): por debajo de qué puerto hace falta ser root para escuchar. Vale 1024. Por eso un contenedor rootless no puede publicar en el 80 del servidor, pero sí en el 8080. Adentro del contenedor sí puede escuchar en 80: el límite es el puerto **del servidor**.
**Qué es `podman pull`:** descargar una imagen. `podman images` las lista; `podman rmi` borra una.
**Qué es `podman search`:** buscar imágenes en un registro por nombre.
**Qué es `podman run`:** crear un contenedor a partir de una imagen y arrancarlo. `-d` en segundo plano; `--name` el nombre; `-p servidor:contenedor` publica un puerto; `-v servidor:contenedor` monta un volumen; `-it ... bash` abre una terminal adentro; `--rm` lo borra al salir.
**Qué es `-p 8080:8080`:** "el puerto 8080 del servidor va al 8080 del contenedor". El primero es afuera; el segundo, adentro. `-p 8083:80` = el 8083 de afuera llega al 80 de adentro.
**Qué es `podman ps` / `ps -a`:** los contenedores que corren / todos, incluidos los detenidos. Un contenedor detenido sigue existiendo hasta que se lo borra con `rm`.
**Qué es `podman logs`:** lo que el programa principal del contenedor escribió en pantalla. Es el `journalctl -u` de los contenedores.
**Qué es `podman exec -it web bash`:** abrir una terminal adentro de un contenedor que ya está corriendo, sin detenerlo.
**Qué es `podman top` / `port` / `inspect` / `stats`:** sus procesos / sus puertos publicados / todos sus datos / CPU y memoria (`--no-stream` = una sola vez, no en vivo).
**Qué es `podman stop` / `start` / `rm -f`:** detener / volver a arrancar / borrar (`-f` aunque esté corriendo).
**Qué es `podman image inspect`:** los datos de una imagen: qué puertos declara (`ExposedPorts`), como qué usuario corre (`User`).
**Qué es `container_t`:** la etiqueta SELinux con la que corren los procesos de un contenedor. Solo puede leer archivos etiquetados para contenedores.
**Qué es `user_home_t`:** la etiqueta de los archivos de tu home. `container_t` no la puede leer: por eso un volumen en el home da `403` hasta que se reetiqueta.
**Qué es `:Z`:** al final del `-v`: "reetiquetá esta carpeta del servidor como `container_file_t` y marcala como de este contenedor". Cambia la etiqueta **en el servidor** (se ve con `ls -Z`). Las dos categorías `c...,c...` son la marca privada de ese contenedor.
**Qué es `:z`:** lo mismo, pero compartido: varios contenedores pueden leer la misma carpeta. Dos contenedores con `:Z` sobre la misma carpeta se pisan la etiqueta: el segundo la roba al primero.
**Qué es `container_file_t`:** la etiqueta que `:Z` y `:z` ponen. Es la única que `container_t` puede leer y escribir.
**Qué es un `Containerfile`:** la receta para construir una imagen propia. `FROM` de qué imagen parto, `RUN` qué comando ejecuto (cada uno es una capa), `EXPOSE` qué puerto declara, `CMD` qué programa arranca. Es el mismo formato que `Dockerfile`.
**Qué es `podman build -t miweb .`:** construir la imagen siguiendo el `Containerfile` de la carpeta actual (el `.`), y llamarla `miweb`. Sin registro delante, queda como `localhost/miweb`.
**Qué es `RUN mkdir -p /run/httpd`:** Apache necesita esa carpeta para su archivo de PID y, según cómo se construya, puede no quedar en la imagen. Se crea a mano para que arranque siempre.
**Qué es `-DFOREGROUND`:** que Apache se quede en primer plano en vez de irse al fondo. En un contenedor, el programa principal tiene que quedarse corriendo: si termina, el contenedor termina.
**Qué es `podman tag`:** ponerle otro nombre a la misma imagen. No copia nada: dos nombres, un solo `IMAGE ID`.
**Qué es `podman image prune`:** borrar imágenes que quedaron sin nombre después de reconstruir. `-f` sin preguntar.
**Qué es `skopeo inspect`:** ver los datos de una imagen que está en un registro sin descargarla: etiquetas disponibles, arquitectura.
**Qué es Quadlet:** la forma actual de convertir un contenedor en un servicio de systemd. Un archivo `nombre.container` en `~/.config/containers/systemd/`; systemd lo lee y genera `nombre.service` solo. Las claves de `[Container]` son las opciones de `podman run` con otro nombre: `Image=`, `PublishPort=` (el `-p`), `Volume=` (el `-v`), `ContainerName=`.
**Qué es `systemctl --user`:** el systemd **del usuario**. `student` tiene su propio systemd con sus propios servicios, sin `sudo`. Arranca cuando el usuario inicia sesión y muere cuando cierra la última, salvo con linger.
**Qué es `default.target`:** el punto de arranque del systemd del usuario. `WantedBy=default.target` = "arrancá esto cuando arranque mi systemd".
**Qué es "transient or generated":** el mensaje de `systemctl --user enable web.service`. Quiere decir "este servicio lo generé yo desde un `.container`, no se habilita a mano". No es un fallo: el `[Install]` del archivo ya lo deja habilitado.
**Qué es `conmon`:** el proceso pequeño que vigila cada contenedor. Es el `Main PID` que muestra `systemctl status`.
**Qué es `quadlet -dryrun -user`:** `/usr/libexec/podman/quadlet -dryrun -user` muestra el `.service` que se va a generar sin generarlo. Si el `.container` tiene un error, acá lo dice.
**Qué es linger:** que el systemd de un usuario arranque **con el sistema** y siga vivo aunque nadie haya iniciado sesión. `sudo loginctl enable-linger student` lo activa; queda un archivo en `/var/lib/systemd/linger/`. Sin esto, el contenedor no vuelve después de un reinicio hasta que alguien entre por SSH.
**Qué es `loginctl`:** la herramienta de systemd para sesiones y usuarios. `loginctl show-user student` muestra, entre otras cosas, `Linger=yes` o `no`.
**Qué es `podman generate systemd --new --files --name miweb`:** la forma vieja: a partir de un contenedor que ya corre, escribe un `.service` normal. `--new` hace que el servicio **cree** el contenedor en cada arranque (`podman run --replace`); `--files` lo escribe a un archivo. Va en `~/.config/systemd/user/` y se habilita con `enable --now`. Está marcada como obsoleta, pero el examen RHCSA la acepta.

---

## Vocabulario y el nombre completo

**Qué decir:** *"la imagen es el plano de la casa; el contenedor es la casa construida. De un plano salen muchas casas. Lo que hay adentro de la casa desaparece cuando la tirás; lo que tiene que quedar, va en un volumen."*
**Qué señalar:** en el diagrama del nombre, las cuatro partes. La frase: *"siempre el nombre completo. `podman pull httpd` a secas pregunta y confunde."*

---

## Podman y Docker

**Qué decir:** *"misma línea de comandos, sin un servicio central como root. Cada contenedor es un proceso tuyo. Por eso vamos a hacer todo como `student`, sin `sudo`: si hacen `sudo podman ps` van a ver otra lista, vacía."*
**Qué señalar:** el límite del 1024: *"adentro puede escuchar en 80; afuera publicamos 8080."*

---

## Ejecutar y el día a día

Leer el diagrama de `podman run` de izquierda a derecha. En la tabla del día a día, señalar tres: `ps -a` (los detenidos existen), `logs` (el `journalctl` de los contenedores) y `exec -it ... bash` (entrar sin detener).

---

## SELinux y volúmenes

**La frase que hay que decir:** *"una carpeta de tu home tiene etiqueta `user_home_t`. El contenedor corre como `container_t` y no puede leerla. Sin `:Z`, 403. Con `:Z`, Podman la reetiqueta y anda. Esto es SELinux protegiendo al servidor del contenedor, no un error."*
**Qué señalar:** `:Z` para uno, `:z` para varios, y nunca sobre `/home` ni `/etc`.

---

## Imagen propia y Quadlet

**Qué decir del `Containerfile`:** *"una receta: de qué parto, qué instalo, qué arranca. Cada línea es una capa."*
**Qué decir de Quadlet:** *"`podman run -d` muere cuando cerrás la sesión y no vuelve al reiniciar. Quadlet es escribir el `podman run` como un archivo de servicio; systemd se encarga. Y linger es lo que hace que arranque sin que nadie entre."*
**Qué señalar:** en el diagrama, `PublishPort=` es el `-p` y `Volume=` es el `-v` con la ruta completa (`/home/student/web`, sin `~`). En la tabla, la fila de `enable`: **no se usa**. Y la de `loginctl enable-linger`, con `sudo`.
