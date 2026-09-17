# Lab — logrotate (comandos)

Yo hago el primero, ellos los demás, foto.

## Parte 1
```bash
sudo touch /var/log/monitor-disco.log
echo "2026-09-16 08:00:00 OK: / al 12%" | sudo tee -a /var/log/monitor-disco.log
echo "2026-09-16 09:00:00 ALERTA: /backups al 91% (umbral 80%)" | sudo tee -a /var/log/monitor-disco.log
sudo wc -l /var/log/monitor-disco.log
grep -v "#" /etc/logrotate.conf
ls /etc/logrotate.d/
systemctl list-timers logrotate.timer --no-pager
```
Quien hizo el Día 7 tendrá más de 2 líneas (el timer `monitor-disco` viene escribiendo): da igual.

## Parte 2
```bash
sudo vim /etc/logrotate.d/monitor
```
```
/var/log/monitor-disco.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    postrotate
        /usr/bin/logger -t logrotate "monitor-disco.log rotado"
    endscript
}
```
```bash
sudo logrotate -d /etc/logrotate.d/monitor
```

## Parte 3
```bash
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor-disco.log*
echo "2026-09-16 10:00:00 OK: / al 11%" | sudo tee -a /var/log/monitor-disco.log
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor-disco.log*
```
Ellos: una línea más con `echo ... | sudo tee -a`, otro `-f`, otro `ls -l`: quedan el activo, `.1`, `.2.gz` y `.3.gz`.
Los sufijos salen numéricos (`.1`, `.2.gz`) y no con fecha porque al invocar el fragmento suelto no se leen los valores generales de `/etc/logrotate.conf` (`dateext`); cuando lo corra el timer, el nombre será `monitor-disco.log-20260917`. Decirlo solo si alguien pregunta por la fecha.
Si a alguien `-f` le da `destination ... already exists`: corrió `sudo logrotate -f /etc/logrotate.conf` (todo el sistema, con `dateext`) en vez del fragmento. Borrar el archivo con fecha y repetir con la ruta del fragmento.

## Parte 4
```bash
sudo grep monitor-disco /var/lib/logrotate/logrotate.status
journalctl -t logrotate -n 3 --no-pager
```
