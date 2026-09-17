# Lab — Primeros contenedores (comandos)

Todo como `student`, sin `sudo` (salvo `dnf`, `ss` y `sysctl`). Si alguien hace `sudo podman`, va a ver otra lista de contenedores, vacía.

## Parte 1 — Instalar y comprobar que somos rootless
```bash
sudo dnf install -y container-tools
podman --version
podman info | grep rootless
podman info | grep graphRoot
grep student /etc/subuid /etc/subgid
```
En tu VM `podman info | grep -i arch` dice `arm64`; en las de ellos, `amd64`. Decirlo si sale.

Si a alguien `grep student /etc/subuid /etc/subgid` no devuelve nada:
```bash
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 student
podman system migrate
```

## Parte 2 — Registros y descarga
```bash
grep unqualified /etc/containers/registries.conf
podman search registry.access.redhat.com/ubi9 --limit 5
podman pull registry.access.redhat.com/ubi9/ubi
podman pull registry.access.redhat.com/ubi9/httpd-24
podman images
```
Decir: "siempre el nombre completo. `podman pull httpd` a secas pregunta en qué registro buscar."

Si la red del aula está lenta, cada uno hace el `pull` mientras seguís hablando: tarda 1 o 2 minutos.

## Parte 3 — Entrar a un contenedor
```bash
podman run -it --rm registry.access.redhat.com/ubi9/ubi bash
```
Adentro:
```
head -2 /etc/os-release
ps aux
id
exit
```
```bash
podman ps -a
```
Decir: "dos procesos en total, `bash` es el PID 1. No hay systemd, no hay nada más: un contenedor es un programa aislado, no una máquina. Y ese `root` de adentro es `student` afuera."

Si la imagen no trae `ps`: `ls /proc | head -3` muestra los mismos dos PIDs.

## Parte 4 — Mirar la imagen web y lanzarla
```bash
podman image inspect registry.access.redhat.com/ubi9/httpd-24 | grep -A 3 ExposedPorts
podman image inspect registry.access.redhat.com/ubi9/httpd-24 | grep User
sudo ss -tlnp | grep 8080
podman run -d --name web -p 8080:8080 registry.access.redhat.com/ubi9/httpd-24
podman ps
curl -sI http://localhost:8080/ | head -3
```
Decir: "la imagen ya está pensada para rootless: escucha en 8080 y corre como el usuario 1001. El `403` es la página de prueba de Apache: todavía no le dimos contenido."

Según la versión de la imagen puede responder `200` con una página propia: el mensaje no cambia.

## Parte 5 — Los comandos de todos los días
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
Decir: "`logs` es el `journalctl` del contenedor; `exec -it` es entrar sin detenerlo."

## Parte 6 — Detener y arrancar
```bash
podman stop web
podman ps
podman ps -a
podman start web
podman ps
```
Decir: "un contenedor detenido sigue existiendo. `ps` muestra los que corren; `ps -a`, todos."

## Parte 7 — El límite del puerto 80
```bash
podman run -d --name web80 -p 80:8080 registry.access.redhat.com/ubi9/httpd-24
podman rm -f web80
sudo sysctl net.ipv4.ip_unprivileged_port_start
```
El texto del error depende de la versión (`permission denied` o `address already in use`); el `rm -f` puede decir `no such container` si el `run` no llegó a crearlo. Decir: "por debajo de 1024, solo root. La práctica es publicar arriba de 1024 y, si hace falta el 80, poner Apache o el firewall adelante. No tocamos el `sysctl`."
