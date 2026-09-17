# Lab — Imagen propia

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

Ahora ustedes: lo mismo. Foto.

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

Todos a la vez (tarda 1 o 2 minutos). Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
localhost/miweb  1.0     abcd1234...
localhost/miweb  latest  abcd1234...
```
Mismo `IMAGE ID`: dos nombres, una imagen. `prune` borra las que quedaron sin nombre (hoy, ninguna).
