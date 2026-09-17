# Lab — El contenedor como servicio (comandos)

Todo como `student`, entrando por SSH directo. Si alguien hizo `sudo -i` y después `su - student`, `systemctl --user` falla con `Failed to connect to bus`: que salga y entre por SSH otra vez.

## Parte 1 — Limpiar
```bash
podman rm -f web web2
podman ps -a
```
Decir: "`web` ocupaba el 8080 que va a usar el servicio, y `web2` tenía `:Z` sobre la misma carpeta `~/web`. Dos `:Z` sobre la misma carpeta se roban la etiqueta entre sí."

## Parte 2 — El archivo Quadlet
```bash
mkdir -p ~/.config/containers/systemd
vim ~/.config/containers/systemd/web.container
```
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
Pegar el texto en el chat. Decir: "`PublishPort` es el `-p`, `Volume` es el `-v`. La ruta va completa: `/home/student/web`, sin `~`. El `dryrun` muestra el servicio que systemd va a generar; el contenedor se va a llamar `systemd-web`."

Si el `dryrun` no muestra `web.service`: el archivo está fuera de `~/.config/containers/systemd/`, no termina en `.container`, o una clave está mal escrita.

## Parte 3 — Arrancar el servicio
```bash
systemctl --user daemon-reload
systemctl --user start web.service
systemctl --user enable web.service
systemctl --user status web.service --no-pager | head -6
podman ps
curl http://localhost:8080/
```
Decir: "el error de `enable` es esperado: el servicio es generado y el `[Install]` ya lo dejó habilitado. Desde ahora, `systemctl --user stop|start|restart web.service` y `journalctl --user -u web.service`. Si hacen `podman stop systemd-web`, `Restart=always` lo vuelve a levantar."

Si `start` falla con `port is already allocated`: no borraron `web` en la Parte 1, o el Apache del servidor quedó en 8080. `sudo ss -tlnp | grep 8080`.

## Parte 4 — Arrancar sin que nadie inicie sesión
```bash
loginctl show-user student | grep Linger
sudo loginctl enable-linger student
loginctl show-user student | grep Linger
ls /var/lib/systemd/linger/
```
Decir: "sin linger, el systemd de `student` muere al cerrar la última sesión SSH y no arranca al reiniciar. Con linger, arranca con el sistema. Esto es lo que hace que el contenedor esté arriba antes de que nadie entre."

## Parte 5 — La forma vieja
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
Decir: "el aviso `DEPRECATED` es real: está en camino de salida, pero el examen la acepta. `--new` hace que el servicio cree el contenedor en cada arranque: por eso pudimos borrarlo. Y acá `enable --now` sí funciona porque es un `.service` normal. Resumen: Quadlet = archivo `.container`, sin `enable`; forma vieja = `.service` generado, con `enable`."

Si `podman generate systemd` ya no existe en la versión instalada: esta parte se explica en voz alta y el material del examen es solo Quadlet.

## Parte 6 — La prueba de verdad
```bash
sudo reboot
```
Esperar un minuto. Todos, en el navegador y **antes de conectarse por SSH**: `http://192.168.56.10:8080/` y `http://192.168.56.10:8083/` (vos con tu IP host-only). Después SSH y:
```bash
systemctl --user is-active web.service container-miweb.service
podman ps
```
Decir: "cargaron sin que nadie iniciara sesión. Eso es Quadlet más linger. Ahora, snapshot `dia09-fin`."

Si a alguien no le carga tras el reboot: `loginctl show-user student | grep Linger` (falta el linger) o `ls /var/lib/systemd/linger/`.
