# Día 9 — Servicios de red y contenedores

## Qué vamos a hacer

1. **Servidor web** — Apache con dos sitios en una sola IP, y HTTPS
2. **NFS y autofs** — Compartir carpetas entre servidores Linux, y que se monten solas al usarlas
3. **Samba y FTP** — Una carpeta compartida que se abre desde Windows, y un FTP con usuarios enjaulados
4. **Podman** — Contenedores sin Docker: imágenes, volúmenes, imagen propia y arranque automático con el sistema

Hasta ayer el servidor se protegía. Hoy el servidor **publica**: cada bloque termina con un servicio al que se entra desde afuera, abierto en el firewall y con SELinux resuelto.

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 8 | **Comandos:** Apache — cómo está armado, virtual host, HTTPS (explicado en consola) |
| 1 | 17 | **Lab 1:** dos sitios en una IP — yo hago el primero, ustedes los demás, foto |
| 1 | 10 | **Lab 2:** HTTPS con certificado propio — yo hago, ustedes repiten, foto |
| 2 | 8 | **Comandos:** NFS y autofs (explicado en consola) |
| 2 | 12 | **Lab 1:** servidor NFS — yo hago el primero, ustedes los demás, foto |
| 2 | 10 | **Lab 2:** cliente NFS y `fstab` — yo hago el primero, ustedes los demás, foto |
| 2 | 15 | **Lab 3:** autofs — yo hago el primero, ustedes los demás, foto |
| 3 | 5 | **Comandos:** Samba y FTP (explicado en consola) |
| 3 | 20 | **Lab 1:** carpeta compartida con Samba — yo hago, ustedes repiten, foto |
| 3 | 10 | **Lab 2:** FTP enjaulado — yo hago, ustedes repiten, foto |
| — | 15 | Descanso |
| 4 | 10 | **Comandos:** Podman — imagen, contenedor, registro, volumen, servicio (explicado en consola) |
| 4 | 25 | **Lab 1:** primeros contenedores — yo hago, ustedes repiten, foto |
| 4 | 15 | **Lab 2:** volúmenes y SELinux — yo hago, ustedes repiten, foto |
| 4 | 10 | **Lab 3:** imagen propia — yo hago, ustedes repiten, foto |
| 4 | 20 | **Lab 4:** el contenedor como servicio — yo hago, ustedes repiten, foto |
| — | 0 | Reto individual — queda de tarea (Ticket #0931) |
| — | 10 | Cierre y snapshot |
| — | 20 | Colchón (margen para imprevistos) |

---

## Al terminar

Publicar un sitio web, una carpeta NFS y una carpeta Samba con el firewall y SELinux bien puestos, y dejar un contenedor que arranca solo con el servidor, sin que nadie inicie sesión.

---

## Tarea

1. **Snapshot `dia09-fin`** con la VM apagada, después de comprobar que tras el reinicio los contenedores están arriba y `ls /remoto/compartido` funciona. **No borrar** contenedores, mapas de autofs ni carpetas compartidas: el Día 10 se rompen a propósito para arreglarlos.
2. **El reto** (Ticket #0931). Mandar por el chat la salida del bloque de Verificación y la captura del navegador.
3. **Espacio en disco:** `df -h /` tiene que mostrar al menos 3G libres (las imágenes ocupan cerca de 1G). Si no: `podman image prune -f` y `sudo dnf clean all`.
