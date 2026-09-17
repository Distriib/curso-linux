# Lab 1 — `cron` (comandos)

Pegar los dos crontabs en el chat.

## Parte 1 — Cómo está `cron` en este servidor
```bash
systemctl is-active crond
ls -d /etc/cron*
crontab -l
```

## Parte 2 — Dos tareas cada minuto
```bash
cat > ~/crontab-prueba <<'EOF'
* * * * * date >> /home/student/cron-prueba.txt
* * * * * hola.sh >> /home/student/cron-hola.txt 2>&1
EOF
crontab ~/crontab-prueba
crontab -l
```
Mirar `date` para saber cuánto falta para el próximo minuto. Mientras, explicar la Parte 3.

## Parte 3 — La trampa del `PATH`
```bash
sudo tail -3 /var/log/cron
cat ~/cron-prueba.txt
cat ~/cron-hola.txt
```
Si `cat` dice `No such file`, todavía no pasó el minuto: esperar y repetir.
Decir: "las dos líneas están en `/var/log/cron`: `cron` ejecutó ambas. La de `hola.sh` falló porque `/bin/sh` no lo encontró. El script no cambió; el entorno sí".

## Parte 4 — El crontab definitivo
```bash
cat > ~/mi-crontab <<'EOF'
SHELL=/bin/bash
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""
30 23 * * *        /home/student/bin/backup.sh /home/student/empresa >> /home/student/backup.log 2>&1
*/15 8-17 * * 1-5  hola.sh >> /home/student/hola.log 2>&1
EOF
crontab ~/mi-crontab
crontab -l
rm -f ~/cron-prueba.txt ~/cron-hola.txt ~/crontab-prueba
```
Ellos:
```bash
echo "0 8 * * 1  /home/student/bin/revisar.sh /home/student/empresa >> /home/student/revisar.log 2>&1" >> ~/mi-crontab
crontab ~/mi-crontab
crontab -l
```

## Parte 5 — El `cron` del sistema
```bash
cat /etc/crontab
cat /etc/cron.d/0hourly
ls /etc/cron.hourly /etc/cron.daily
```

## Parte 6 — Quién puede usar `crontab`
```bash
sudo chage -d "$(date +%F)" jperez
cat /etc/cron.deny
echo jperez | sudo tee /etc/cron.deny
sudo -u jperez crontab -l
sudo truncate -s 0 /etc/cron.deny
sudo -u jperez crontab -l
```
Si aun así sale `not allowed to access to (crontab) because of pam configuration`: `sudo passwd -S jperez` y `sudo chage -l jperez`; el `chage -d` de arriba tiene que haber quedado con la fecha de hoy.
