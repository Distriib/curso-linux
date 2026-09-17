# Lab — Imagen propia (comandos)

## Parte 1 — El `Containerfile`
```bash
mkdir -p ~/miweb
cd ~/miweb
vim Containerfile
```
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
```
Pegar el texto en el chat. Decir, línea por línea: "de qué parto; qué instalo; la carpeta que Apache necesita al arrancar; la página; qué puerto declara; qué programa arranca y se queda en primer plano."

## Parte 2 — Construir
```bash
podman build -t miweb .
podman images
```
Tarda 1 o 2 minutos por el `dnf install`. Mientras corre: "cada instrucción es una capa; si mañana cambio solo la página, las capas anteriores se reutilizan".

Si a alguien falla en `dnf install`: sin red o repos UBI inaccesibles. Reintentar; si no, seguir el resto del día con `ubi9/httpd-24` y hacer este lab de tarea.

## Parte 3 — Ejecutarla
```bash
podman run -d --name miweb -p 8083:80 miweb
curl http://localhost:8083/
cd ~
```
Decir: "adentro escucha en el 80 porque adentro el proceso es root de su propio mundo. Afuera publicamos 8083."

Si el contenedor arranca y muere (`podman ps -a` lo muestra `Exited`): `podman logs miweb`. Si dice `could not create /run/httpd/httpd.pid`, falta la línea `RUN mkdir -p /run/httpd`: agregarla y reconstruir.

## Parte 4 — Etiquetas y limpieza
```bash
podman tag miweb miweb:1.0
podman images | grep miweb
podman image prune -f
```
Decir: "`tag` no copia nada: dos nombres, un solo `IMAGE ID`. `prune` borra las que quedaron sin nombre; hoy no hay ninguna."
