# Lab — Construir `web01` (comandos)

Yo hago el primero, ellos los demás, foto. Contraseña de las cuentas nuevas: `Pgn.2026`. Al terminar, **snapshot `dia10-pre-romper`** en todas las VMs antes de pasar al reto.

## Parte 1
```bash
sudo hostnamectl set-hostname web01.lab.local
hostname
hostnamectl --static
```

## Parte 2
```bash
getent group sistemas
sudo useradd -m -c "Desarrollador dev01" -G sistemas dev01
echo 'Pgn.2026' | sudo passwd --stdin dev01
sudo mkdir -p /srv/compartido
sudo chown root:sistemas /srv/compartido
sudo chmod 2770 /srv/compartido
```
Ellos:
```bash
sudo useradd -m -c "Desarrollador dev02" -G sistemas dev02
echo 'Pgn.2026' | sudo passwd --stdin dev02
```
Comprobar:
```bash
id dev01
id dev02
ls -ld /srv/compartido
sudo -u dev01 touch /srv/compartido/prueba-dev01
ls -l /srv/compartido
```
Si a alguien `getent group sistemas` no le devuelve nada (no hizo el Día 3): `sudo groupadd -g 3001 sistemas` y seguir.

## Parte 3
```bash
grep DocumentRoot /etc/httpd/conf.d/00-default.conf
echo "<h1>Servidor web01 - Procuraduria General de la Nacion</h1>" | sudo tee /var/www/html/index.html
ls -Z /var/www/html/index.html
sudo systemctl enable --now httpd
curl -s http://localhost/
```
Si el `grep` no devuelve `DocumentRoot /var/www/html` (no hizo el Día 9): `sudo vim /etc/httpd/conf.d/00-default.conf` con este contenido, después `sudo apachectl configtest` y `sudo systemctl restart httpd`:
```
<VirtualHost *:80>
    ServerName web01.lab.local
    ServerAlias web01 rhel01 localhost
    DocumentRoot /var/www/html
</VirtualHost>
```
Si `ls -Z` no dice `httpd_sys_content_t`: `sudo restorecon -v /var/www/html/index.html`.

## Parte 4
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --zone=internal --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
```
Ellos: navegador, `http://192.168.56.10/` y `http://localhost:8080/`. En tu Mac, la IP host-only de tu UTM.

## Parte 5
```bash
findmnt /backups
sudo cp /home/student/bin/backup.sh /usr/local/bin/backup.sh
sudo chmod 755 /usr/local/bin/backup.sh
sudo vim /etc/cron.d/backup-empresa
```
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
Si `findmnt /backups` no devuelve nada (no hizo el reto del Día 6), crear el volumen antes de seguir:
```bash
sudo lvcreate -n lv_backups -L 1G vg_datos
sudo mkfs.ext4 /dev/vg_datos/lv_backups
sudo mkdir -p /backups
echo "/dev/mapper/vg_datos-lv_backups  /backups  ext4  defaults,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
findmnt /backups
```
Quien no tenga `vg_datos` no puede hacer T3: que lo salte en el reto.
Si no existe `/home/student/bin/backup.sh` (no hizo el Día 7): pegarle el script del Lab 2.2 de la guía del Día 7 en `/usr/local/bin/backup.sh` con `sudo vim`.

## Parte 6
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
Todo bien en todas las VMs → snapshot `dia10-pre-romper`. No seguir hasta que los cuatro lo tengan.
