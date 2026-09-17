# Reto — Solución

Es tarea: no se hace en clase. Corregir con el bloque de Verificación que ellos pegan en el chat.

## 1. Contenido
```bash
mkdir -p ~/portal
echo "<h1>Portal PGN</h1>" > ~/portal/index.html
```

## 2. Quadlet
```bash
mkdir -p ~/.config/containers/systemd
vim ~/.config/containers/systemd/portal.container
```
```
[Unit]
Description=Portal PGN en contenedor

[Container]
Image=registry.access.redhat.com/ubi9/httpd-24
ContainerName=portal
PublishPort=8085:8080
Volume=/home/student/portal:/var/www/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```
```bash
systemctl --user daemon-reload
systemctl --user start portal.service
systemctl --user is-active portal.service
podman ps
curl http://localhost:8085/
```

## 3. Firewall y arranque sin sesión
```bash
sudo firewall-cmd --add-port=8085/tcp --permanent
sudo firewall-cmd --zone=internal --add-port=8085/tcp --permanent
sudo firewall-cmd --reload
sudo loginctl enable-linger student
```

## 4. NFS solo lectura hacia la red host-only
El home de `student` es `700`: para que el servidor NFS pueda recorrer la ruta hasta `portal`, el home tiene que dejar pasar (`711` = los demás pueden entrar pero no listar).
```bash
sudo chmod 711 /home/student
echo "/home/student/portal   192.168.56.0/24(ro,sync)" | sudo tee -a /etc/exports
sudo exportfs -rav
showmount -e localhost
```

## 5. autofs (el mapa indirecto `/remoto` ya existe)
```bash
echo "portal   -ro   192.168.56.10:/home/student/portal" | sudo tee -a /etc/auto.remoto
sudo systemctl reload autofs
cat /remoto/portal/index.html
mount | grep portal
```

## Verificación
```bash
sudo reboot
```
Desde la computadora propia, sin SSH: `http://192.168.56.10:8085/` → `Portal PGN`. Después `ssh -p 2222 student@localhost` y:
```bash
systemctl --user is-active portal.service
podman ps
curl http://localhost:8085/
loginctl show-user student | grep Linger
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
showmount -e localhost
cat /remoto/portal/index.html
mount | grep portal
```

## Errores que se ven

- Sin `loginctl enable-linger`: todo funciona hasta el reinicio; después del reboot el navegador no carga hasta que alguien entra por SSH.
- `PublishPort=8085:80` en vez de `8085:8080`: la imagen escucha en 8080; `curl` da `connection reset` o no responde.
- Volumen sin `:Z`: `403 Forbidden`; `sudo ausearch -m AVC -ts recent` lo muestra.
- `Volume=~/portal:...`: Quadlet no expande `~`; hay que escribir `/home/student/portal`.
- Puerto 8085 abierto solo en `public`: carga en `localhost:8085` (si agregó la regla NAT) y no desde `192.168.56.10`.
- Espacio antes del paréntesis en `/etc/exports`: exporta a todo el mundo; `exportfs -v` lo delata con `<world>`.
- Falta `systemctl reload autofs` después de editar `auto.remoto`: `ls /remoto/portal` da `No such file or directory`.
- `ls /remoto/portal` da `Permission denied`: el home sigue en `700`, o exportó `~/portal` literal en vez de `/home/student/portal`.
- Hizo el contenedor con `sudo podman`: `podman ps` como `student` no lo muestra y `systemctl --user` no lo encuentra. Todo como `student`, sin `sudo`.
