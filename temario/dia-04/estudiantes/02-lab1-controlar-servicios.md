# Lab 1 — Controlar servicios

Vamos a leer el estado de los servicios, instalar el servidor web `httpd` (Apache) y administrarlo de principio a fin: arrancar, habilitar, recargar, prohibir. Y ver qué hace cada comando por debajo.

---

## Parte 1 — Panorama

**¿Qué corre ahora, qué arranca con el sistema, y qué falló?**

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled | head -15
systemctl --failed
```

Foto.

**Comprobar:**
```
  UNIT                     LOAD   ACTIVE SUB     DESCRIPTION
  auditd.service           loaded active running Security Auditing Service
  chronyd.service          loaded active running NTP client/server
  crond.service            loaded active running Command Scheduler
  sshd.service             loaded active running OpenSSH server daemon
  ...
UNIT FILE                  STATE   PRESET
auditd.service             enabled enabled
chronyd.service            enabled enabled
  ...
0 loaded units listed.
```
`list-units` habla del **ahora**; `list-unit-files` habla del **arranque**. `--failed` vacío es lo deseable.

---

## Parte 2 — Radiografía de `sshd`

**¿De qué archivo sale `sshd`, y qué hace exactamente `reload`?**

```bash
systemctl status sshd
systemctl cat sshd
systemctl is-active sshd
systemctl is-enabled sshd
```

Ahora ustedes: lo mismo con `chronyd`. Foto.

**Comprobar:**
```
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 890 (sshd)
...
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
After=network.target sshd-keygen.target
...
[Service]
Type=notify
ExecStart=/usr/sbin/sshd -D ...
ExecReload=/bin/kill -HUP ...
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
active
enabled
```
`ExecReload` es un `kill -HUP` al proceso principal: eso es "reload". `KillMode=process` es por lo que un `restart sshd` no corta tu sesión.

---

## Parte 3 — Instalar Apache

**¿Instalar un servicio lo arranca?**

```bash
sudo dnf install -y httpd
systemctl status httpd
```

Foto.

**Comprobar:**
```
...
Complete!
○ httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; preset: disabled)
     Active: inactive (dead)
```
`disabled` e `inactive`: instalado no significa nada más.

---

## Parte 4 — Arrancarlo y ver que **no** quedó habilitado

```bash
sudo systemctl start httpd
systemctl is-active httpd
systemctl is-enabled httpd
echo "<h1>rhel01 - Dia 04</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

Foto.

**Comprobar:**
```
active
disabled
<h1>rhel01 - Dia 04</h1>
<h1>rhel01 - Dia 04</h1>
```
Funciona, pero un reinicio lo dejaría apagado. ¿Qué comando falta?

---

## Parte 5 — Habilitar, y ver qué hizo `enable` en realidad

```bash
sudo systemctl enable httpd
ls -l /etc/systemd/system/multi-user.target.wants/httpd.service
systemctl is-enabled httpd
```

Foto.

**Comprobar:**
```
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
lrwxrwxrwx. 1 root root 37 ... /etc/systemd/system/multi-user.target.wants/httpd.service -> /usr/lib/systemd/system/httpd.service
enabled
```
Un enlace, nada más. Cuando `multi-user.target` arranca, arranca todo lo que hay en su carpeta `.wants/`.

---

## Parte 6 — `reload` mantiene el PID, `restart` lo cambia

**¿Cuál uso después de cambiar la configuración de un servicio en producción?**

```bash
systemctl show -p MainPID --value httpd
sudo systemctl reload httpd
systemctl show -p MainPID --value httpd
sudo systemctl restart httpd
systemctl show -p MainPID --value httpd
```

Foto.

**Comprobar:**
```
4321
4321
4410
```
`reload`: mismo PID, los clientes no se enteran. `restart`: PID nuevo, conexiones cortadas.

---

## Parte 7 — Prohibir con `mask`

**¿Qué hago con un servicio que no debe correr bajo ningún concepto?**

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

Foto.

**Comprobar:**
```
Created symlink /etc/systemd/system/httpd.service → /dev/null.
Failed to start httpd.service: Unit httpd.service is masked.
lrwxrwxrwx. 1 root root 9 ... /etc/systemd/system/httpd.service -> /dev/null
masked
Removed "/etc/systemd/system/httpd.service".
active
enabled
```
El `mask` es un enlace a `/dev/null` en `/etc/systemd/system/`, que gana sobre `/usr/lib/systemd/system/`.

---

## Parte 8 — Targets y timers

**¿Cómo se decide en qué "modo" arranca el servidor?**

```bash
systemctl get-default
sudo systemctl set-default graphical.target
sudo systemctl set-default multi-user.target
ls -l /etc/systemd/system/default.target
systemctl list-timers
```

Foto.

**Comprobar:**
```
multi-user.target
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/graphical.target.
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/multi-user.target.
lrwxrwxrwx. 1 root root 41 ... /etc/systemd/system/default.target -> /usr/lib/systemd/system/multi-user.target
NEXT                        LEFT       LAST  PASSED  UNIT                         ACTIVATES
...                                                  dnf-makecache.timer          dnf-makecache.service
...                                                  logrotate.timer              logrotate.service
```
`default.target` es otro enlace. `logrotate` corre por **timer**, no por cron: lo vemos en el Bloque 3.
