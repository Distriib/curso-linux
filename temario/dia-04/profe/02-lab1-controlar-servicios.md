# Lab — Controlar servicios (comandos)

Yo hago el primero, ellos los demás, foto. Si algún `systemctl` sin `sudo` dice `Interactive authentication required`, falta el `sudo`.

## Parte 1 — Panorama
```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled | head -15
systemctl --failed
```
Si la primera salida queda en el visor: `q`.

## Parte 2 — Radiografía de `sshd`
```bash
systemctl status sshd
systemctl cat sshd
systemctl is-active sshd
systemctl is-enabled sshd
```
Ellos lo repiten con `chronyd`:
```bash
systemctl status chronyd
systemctl cat chronyd
systemctl is-active chronyd
systemctl is-enabled chronyd
```
Qué decir sobre `sshd`: "`ExecReload` es un `kill -HUP`: eso es todo lo que hace `reload`. `KillMode=process` es por lo que un `restart sshd` no nos tira de la sesión. Igual: **nunca `stop sshd` por SSH**."

## Parte 3 — Instalar Apache
```bash
sudo dnf install -y httpd
systemctl status httpd
```
Si `dnf` dice `This system is not registered` o `no enabled repositories`: `sudo subscription-manager register` y repetir. Sin registro, nada del Bloque 4 funciona: resolverlo ahora.

## Parte 4 — Arrancarlo y ver que no quedó habilitado
```bash
sudo systemctl start httpd
systemctl is-active httpd
systemctl is-enabled httpd
echo "<h1>rhel01 - Dia 04</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```
Respuesta a la pregunta: falta `enable`. Desde el navegador de la máquina propia no carga: el firewall de la VM solo deja pasar SSH y cockpit; se abre el Día 8.

## Parte 5 — Habilitar
```bash
sudo systemctl enable httpd
ls -l /etc/systemd/system/multi-user.target.wants/httpd.service
systemctl is-enabled httpd
```

## Parte 6 — `reload` vs `restart`
```bash
systemctl show -p MainPID --value httpd
sudo systemctl reload httpd
systemctl show -p MainPID --value httpd
sudo systemctl restart httpd
systemctl show -p MainPID --value httpd
```
Respuesta: `reload`. Si un servicio no tiene `ExecReload`, `systemctl reload` dice `Job type reload is not applicable`.

## Parte 7 — `mask`
```bash
sudo systemctl stop httpd
sudo systemctl mask httpd
sudo systemctl start httpd
ls -l /etc/systemd/system/httpd.service
systemctl is-enabled httpd
sudo systemctl unmask httpd
sudo systemctl enable --now httpd
systemctl is-active httpd
systemctl is-enabled httpd
```
Qué decir: "esto es exactamente el ticket 2 del reto. Guárdenlo."

## Parte 8 — Targets y timers
```bash
systemctl get-default
sudo systemctl set-default graphical.target
sudo systemctl set-default multi-user.target
ls -l /etc/systemd/system/default.target
systemctl list-timers
```
Qué señalar en `list-timers`: `logrotate.timer` y `dnf-makecache.timer`. "En RHEL 9 estas dos cosas corren por timer, no por cron."
