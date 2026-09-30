# Día 9 — Servicios de red y contenedores

## Qué vamos a hacer

1. **Servidor web** — Apache con dos sitios en una sola IP, y HTTPS
2. **NFS y autofs** — Compartir carpetas entre servidores Linux, y que se monten solas al usarlas
3. **Samba y FTP** — Una carpeta compartida que se abre desde Windows, y un FTP con usuarios enjaulados
4. **Podman** — Contenedores sin Docker: imágenes, volúmenes, imagen propia y arranque automático con el sistema

Hasta ayer el servidor se protegía. Hoy el servidor **publica**: cada bloque termina con un servicio al que se entra desde afuera, abierto en el firewall y con SELinux resuelto.

## Los laboratorios del día

| Lab | Qué se hace |
|---|---|
| 1.1 | Dos sitios en una misma IP y puerto, con virtual hosts |
| 1.2 | HTTPS con un certificado generado en la propia VM |
| 2.1 | Servidor NFS: exportar dos carpetas con permisos distintos |
| 2.2 | Cliente NFS: montar a mano y dejarlo en `/etc/fstab` |
| 2.3 | autofs: que se monte solo al entrar y se suelte solo al dejar de usarlo |
| 3.1 | Carpeta compartida con Samba, abierta desde Linux y desde Windows |
| 3.2 | FTP con cada usuario enjaulado en su propia carpeta |
| 4.1 | Primeros contenedores con Podman: imágenes, `run`, `exec`, `logs` |
| 4.2 | Volúmenes: servir una carpeta del servidor, y el `:Z` de SELinux |
| 4.3 | Construir una imagen propia con un `Containerfile` |
| 4.4 | El contenedor como servicio, que arranca solo con la máquina |

Cada lab termina con una sección **Solución** con todos los comandos seguidos.

**Reto del día:** Ticket #0931 — el portal en contenedor, arrancando solo, exportado por NFS y montado con autofs.

## Antes de empezar

El Día 9 arranca con el servidor tal como lo dejó la tarea del Día 8: Apache en el puerto 80 sirviendo `/var/www/html`. Si `curl http://localhost/` no devuelve la página, el Lab 1.1 Parte 1 explica cómo arreglarlo antes de seguir.

Hace falta **espacio en disco**: las imágenes de contenedor del Bloque 4 ocupan cerca de 1 GB. Comprobar con `df -h /` que haya al menos 3 GB libres; si no, `podman image prune -f` y `sudo dnf clean all`.

---

## Al terminar

Publicar un sitio web, una carpeta NFS y una carpeta Samba con el firewall y SELinux bien puestos, y dejar un contenedor que arranca solo con el servidor, sin que nadie inicie sesión.

---

## Para cerrar el día

1. **El reto** (Ticket #0931), que junta el contenedor, NFS y autofs en un solo ejercicio.
2. **Snapshot `dia09-fin`** con la VM apagada, después de comprobar que tras el reinicio los contenedores están arriba y `ls /remoto/compartido` funciona.
3. **Espacio en disco:** `df -h /` tiene que mostrar al menos 3G libres. Si no: `podman image prune -f` y `sudo dnf clean all`.
