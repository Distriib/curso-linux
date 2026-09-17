# Lab 4.1 — Construir `web01`

Vamos a dejar la VM como el servidor `web01` del reto, verificarlo pieza por pieza y tomar el snapshot `dia10-pre-romper`. Contraseña de las cuentas nuevas: **`Pgn.2026`**.

| Qué | Valor |
|---|---|
| Nombre | `web01.lab.local` |
| Usuarios | `dev01`, `dev02` — grupo suplementario `sistemas` |
| Carpeta compartida | `/srv/compartido` — `root:sistemas`, `2770` |
| Sitio | `/var/www/html/index.html` con `<h1>Servidor web01 - Procuraduria General de la Nacion</h1>` |
| Respaldos | `/backups` (LV `lv_backups`) · `/usr/local/bin/backup.sh` · cron a las 23:30 |

---

## Parte 1 — El nombre

**¿Cómo cambio el nombre del servidor de forma permanente?**

```bash
sudo hostnamectl set-hostname web01.lab.local
hostname
hostnamectl --static
```

**Comprobar:** `web01.lab.local` dos veces. El prompt cambia al abrir la próxima sesión.

---

## Parte 2 — Las personas y la carpeta compartida

**¿Cómo doy de alta a dos desarrolladores que comparten una carpeta del área?**

```bash
getent group sistemas
sudo useradd -m -c "Desarrollador dev01" -G sistemas dev01
echo 'Pgn.2026' | sudo passwd --stdin dev01
sudo mkdir -p /srv/compartido
sudo chown root:sistemas /srv/compartido
sudo chmod 2770 /srv/compartido
```

Ahora ustedes: `dev02`, igual, con descripción `Desarrollador dev02`. Foto.

**Comprobar:**
```bash
id dev01
id dev02
ls -ld /srv/compartido
sudo -u dev01 touch /srv/compartido/prueba-dev01
ls -l /srv/compartido
```
```
sistemas:x:3001:ana,carlos
uid=...(dev01) gid=...(dev01) groups=...(dev01),3001(sistemas)
uid=...(dev02) gid=...(dev02) groups=...(dev02),3001(sistemas)
drwxrws---. 2 root sistemas ... /srv/compartido
-rw-r--r--. 1 dev01 sistemas ... prueba-dev01
```
El archivo de `dev01` nace con grupo `sistemas`: eso es la `s` del setgid (Día 3).

---

## Parte 3 — El sitio

**¿Cómo dejo el sitio de `web01` publicado, con el contexto correcto?**

```bash
grep DocumentRoot /etc/httpd/conf.d/00-default.conf
echo "<h1>Servidor web01 - Procuraduria General de la Nacion</h1>" | sudo tee /var/www/html/index.html
ls -Z /var/www/html/index.html
sudo systemctl enable --now httpd
curl -s http://localhost/
```

**Comprobar:**
```
    DocumentRoot /var/www/html
<h1>Servidor web01 - Procuraduria General de la Nacion</h1>
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html
<h1>Servidor web01 - Procuraduria General de la Nacion</h1>
```
El archivo nace con `httpd_sys_content_t` porque está dentro de `/var/www`: no hace falta ninguna regla extra.

---

## Parte 4 — El firewall, en las dos zonas

**¿Por qué "funciona desde localhost:8080" no alcanza?**

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --zone=internal --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
```

Ahora ustedes: abran en el navegador de su computadora `http://192.168.56.10/` y `http://localhost:8080/`. Foto de las dos.

**Comprobar:**
```
success
success
success
cockpit dhcpv6-client http ssh
cockpit dhcpv6-client http mdns samba-client ssh
```
(si `http` ya estaba, avisa `ALREADY_ENABLED`: está bien.) Las dos listas tienen `http`. El navegador muestra `Servidor web01 - Procuraduria General de la Nacion` por las dos direcciones.

---

## Parte 5 — Los respaldos

**¿Dónde van los respaldos, con qué script, y quién lo corre cada noche?**

```bash
findmnt /backups
sudo cp /home/student/bin/backup.sh /usr/local/bin/backup.sh
sudo chmod 755 /usr/local/bin/backup.sh
sudo vim /etc/cron.d/backup-empresa
```
Contenido:
```
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
30 23 * * * root /usr/local/bin/backup.sh /home/student/empresa /backups >> /var/log/backup.log 2>&1
```
```bash
sudo /usr/local/bin/backup.sh /home/student/empresa /backups
ls -l /backups
df -h /backups
```

**Comprobar:**
```
TARGET   SOURCE                           FSTYPE OPTIONS
/backups /dev/mapper/vg_datos-lv_backups  ext4   rw,...
2026-09-16 ... OK: creado /backups/empresa-2026-09-16-....tar.gz (...K)
2026-09-16 ... Retención: 0 respaldo(s) de más de 7 días eliminado(s)
-rw-r--r--. 1 root root ... empresa-2026-09-16-....tar.gz
/dev/mapper/vg_datos-lv_backups  1.5G  ...M  1.3G   1% /backups
```
El cron del sistema (`/etc/cron.d/`) lleva el **usuario** (`root`) antes del comando; el de `crontab -e` (Día 7) no.

---

## Parte 6 — Verificar todo y snapshot

**¿Cómo confirmo que `web01` está completo antes de romperlo?**

```bash
hostname
systemctl is-active httpd; systemctl is-enabled httpd
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
curl -s http://localhost/
ls -Z /var/www/html/index.html
ls -ld /srv/compartido
sudo -u dev02 touch /srv/compartido/prueba-dev02; ls -l /srv/compartido
sudo passwd -S dev02
getent passwd dev02
df -h /backups
sudo /usr/local/bin/backup.sh /home/student/empresa /backups
cat /etc/cron.d/backup-empresa
```

**Comprobar:** `web01.lab.local` · `active` · `enabled` · `http` en las dos listas · el `<h1>` · `httpd_sys_content_t` · `drwxrws---. 2 root sistemas` · `prueba-dev02` con grupo `sistemas` · `dev02 PS` · `/bin/bash` al final · `/backups` con poco uso · `OK: creado` · las tres líneas del cron.

Todo bien → **snapshot `dia10-pre-romper`** (en VirtualBox, con la VM encendida está bien). Es la red de seguridad del reto: quien se trabe más de 15 minutos en un ticket puede restaurar y seguir con el siguiente.
