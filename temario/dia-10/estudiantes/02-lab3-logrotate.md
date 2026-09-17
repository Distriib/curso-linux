# Lab 2.3 — logrotate: que los logs no llenen el disco

Vamos a rotar `/var/log/monitor-disco.log` (el del Día 7) a diario, conservando **7 copias** comprimidas.

---

## Parte 1 — Cómo está el log y quién rota

**¿Quién decide cuándo se rota un log, y con qué reglas?**

```bash
sudo touch /var/log/monitor-disco.log
echo "2026-09-16 08:00:00 OK: / al 12%" | sudo tee -a /var/log/monitor-disco.log
echo "2026-09-16 09:00:00 ALERTA: /backups al 91% (umbral 80%)" | sudo tee -a /var/log/monitor-disco.log
sudo wc -l /var/log/monitor-disco.log
grep -v "#" /etc/logrotate.conf
ls /etc/logrotate.d/
systemctl list-timers logrotate.timer --no-pager
```

**Comprobar:**
```
N /var/log/monitor-disco.log          (2 o más)
weekly
rotate 4
create
dateext
include /etc/logrotate.d
btmp  chrony  dnf  firewalld  httpd  ...  rsyslog  ...  wtmp
NEXT                        LEFT      LAST   ...   UNIT             ACTIVATES
... 00:00:00 ...            ...       ...          logrotate.timer  logrotate.service
```
`logrotate.conf` pone los valores generales (semanal, 4 copias, sufijo con fecha); cada archivo de `logrotate.d/` los cambia para sus logs. El timer corre a medianoche.

---

## Parte 2 — La política

**¿Cómo le digo a logrotate qué hacer con mi log?**

```bash
sudo vim /etc/logrotate.d/monitor
```
Contenido:
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

**Comprobar** (entre lo que imprime):
```
reading config file /etc/logrotate.d/monitor
rotating pattern: /var/log/monitor-disco.log  after 1 days (7 rotations)
considering log /var/log/monitor-disco.log
  ...
  log does not need rotating ...
```
`-d` **no hace nada**: explica qué haría y por qué. Es la forma segura de validar. (Si dice `log needs rotating`, tampoco rotó: solo lo anuncia.)

---

## Parte 3 — Forzar dos rotaciones y ver la secuencia

**¿Qué pasa con las copias en cada rotación?**

```bash
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor-disco.log*
echo "2026-09-16 10:00:00 OK: / al 11%" | sudo tee -a /var/log/monitor-disco.log
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor-disco.log*
```

Ahora ustedes: agreguen una línea más y roten una tercera vez. ¿Cuántos archivos hay y cuáles están comprimidos? Foto.

**Comprobar:**
```
-rw-r-----. 1 root root    0 ... /var/log/monitor-disco.log
-rw-r--r--. 1 root root  ... ... /var/log/monitor-disco.log.1

-rw-r-----. 1 root root    0 ... /var/log/monitor-disco.log
-rw-r-----. 1 root root  ... ... /var/log/monitor-disco.log.1
-rw-r--r--. 1 root root  ... ... /var/log/monitor-disco.log.2.gz
```
La primera rotación solo renombra a `.1` (`delaycompress`); la segunda corre `.1` a `.2` y **ahí** lo comprime. El activo nace vacío y con `0640` (`create`).

---

## Parte 4 — El rastro

**¿Cómo sé que rotó, y cuándo?**

```bash
sudo grep monitor-disco /var/lib/logrotate/logrotate.status
journalctl -t logrotate -n 3 --no-pager
```

**Comprobar:**
```
"/var/log/monitor-disco.log" 2026-9-16-...
... rhel01 logrotate[NNNN]: monitor-disco.log rotado
... rhel01 logrotate[NNNN]: monitor-disco.log rotado
```
El `postrotate` es donde va `systemctl reload SERVICIO` cuando el programa mantiene el archivo abierto (el fantasma del Lab 1.1).
