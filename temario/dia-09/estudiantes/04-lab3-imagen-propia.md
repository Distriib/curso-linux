# Lab 4.3 — Imagen propia

Vamos a construir una imagen a partir de UBI, con Apache instalado y nuestra página adentro, y a ejecutarla.

| Qué | Valor |
|---|---|
| Carpeta de trabajo | `~/miweb` |
| Imagen | `miweb` (queda como `localhost/miweb`) |
| Contenedor | `miweb`, puerto `8083` del servidor → `80` adentro |

---

## Parte 1 — El `Containerfile`

**¿Qué hace cada línea?**

```bash
mkdir -p ~/miweb
cd ~/miweb
vim Containerfile
```
Pegar:
```
FROM registry.access.redhat.com/ubi9/ubi
RUN dnf -y install httpd && dnf clean all
RUN mkdir -p /run/httpd
RUN echo "<h1>Imagen miweb construida en rhel01</h1>" > /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

**Comprobar:**
```bash
cat Containerfile
```
Seis líneas, tal cual.

---

## Parte 2 — Construir

**¿Cuántas capas tiene la imagen?**

```bash
podman build -t miweb .
podman images
```

(Tarda 1 o 2 minutos.)

**Comprobar:**
```
STEP 1/6: FROM registry.access.redhat.com/ubi9/ubi
STEP 2/6: RUN dnf -y install httpd && dnf clean all
...
Complete!
STEP 3/6: RUN mkdir -p /run/httpd
STEP 4/6: RUN echo ...
STEP 5/6: EXPOSE 80
STEP 6/6: CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
COMMIT miweb
Successfully tagged localhost/miweb:latest
REPOSITORY                                TAG     IMAGE ID      CREATED         SIZE
localhost/miweb                           latest  ...           ... seconds ago  ... MB
registry.access.redhat.com/ubi9/httpd-24  latest  ...
registry.access.redhat.com/ubi9/ubi       latest  ...
```
Cada instrucción es una capa. Sin registro delante, queda como `localhost/miweb`.

---

## Parte 3 — Ejecutarla

**Adentro escucha en el 80. ¿Por qué acá sí se puede?**

```bash
podman run -d --name miweb -p 8083:80 miweb
curl http://localhost:8083/
cd ~
```

**Comprobar:**
```
<h1>Imagen miweb construida en rhel01</h1>
```
El 80 es **adentro** del contenedor, donde el proceso es root. Afuera publicamos el 8083.

---

## Parte 4 — Etiquetas y limpieza

**Si le pongo otro nombre, ¿ocupa el doble?**

```bash
podman tag miweb miweb:1.0
podman images | grep miweb
podman image prune -f
```

**Comprobar:**
```
localhost/miweb  1.0     abcd1234...
localhost/miweb  latest  abcd1234...
```
Mismo `IMAGE ID`: dos nombres, una imagen. `prune` borra las que quedaron sin nombre (hoy, ninguna).

---

# Solución — todos los comandos

```bash
# Parte 1 — el Containerfile
mkdir -p ~/miweb
cd ~/miweb
vim Containerfile
```
Contenido:
```
FROM registry.access.redhat.com/ubi9/ubi
RUN dnf -y install httpd && dnf clean all
RUN mkdir -p /run/httpd
RUN echo "<h1>Imagen miweb construida en rhel01</h1>" > /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```
```bash
cat Containerfile

# Parte 2 — construir (tarda 1 o 2 minutos)
podman build -t miweb .
podman images

# Parte 3 — ejecutarla
podman run -d --name miweb -p 8083:80 miweb
curl http://localhost:8083/
cd ~

# Parte 4 — etiquetas y limpieza
podman tag miweb miweb:1.0
podman images | grep miweb
podman image prune -f
```

**Qué hace cada línea del `Containerfile`:**

| Línea | Qué hace |
|---|---|
| `FROM` | de qué imagen se parte |
| `RUN` | un comando que se ejecuta **al construir**, y su resultado queda grabado en la imagen |
| `EXPOSE` | documenta en qué puerto escucha; **no abre nada** |
| `CMD` | qué se ejecuta **al arrancar** cada contenedor |

**`RUN` contra `CMD`** es la confusión más común: `RUN` corre una sola vez, al construir; `CMD` corre cada vez que alguien hace `podman run`.

**El `-DFOREGROUND` es obligatorio.** Un contenedor vive mientras viva su proceso PID 1. Si Apache arrancara como demonio (yéndose al fondo), el PID 1 terminaría y el contenedor moriría al instante. Por eso todo servicio en contenedor se lanza en primer plano.

**El `mkdir -p /run/httpd`** está porque la imagen UBI pelada no trae esa carpeta y Apache se niega a arrancar sin ella.

**Cada instrucción es una capa,** y por eso `dnf clean all` va pegado al `install` con `&&` en la **misma** línea: en líneas separadas, la capa del `install` ya habría guardado la caché y limpiarla después no achica nada.

**`localhost/miweb`**, sin registro adelante, porque se construyó acá y no se subió a ningún lado.

**Adentro el 80 sí se puede** (el proceso es root dentro de su propio espacio de nombres); afuera se publica en el 8083, que está arriba de 1024.

**`podman tag` no duplica nada:** las dos etiquetas apuntan al mismo `IMAGE ID`. Son dos nombres para la misma imagen.
