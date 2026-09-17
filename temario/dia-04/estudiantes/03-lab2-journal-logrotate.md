# Lab 2 — Journal persistente y rotación

Vamos a dejar el journal en disco para que sobreviva al reinicio, y a configurar la rotación de `/var/log/monitor.log` para que no crezca sin límite.

---

## Parte 1 — Cómo está el journal ahora

**¿Cuántos arranques tiene guardados el journal?**

```bash
ls -ld /var/log/journal
journalctl --disk-usage
journalctl --list-boots
grep Storage /etc/systemd/journald.conf
```

Foto.

**Comprobar:**
```
ls: cannot access '/var/log/journal': No such file or directory
Archived and active journals take up 16.0M in the file system.
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
  0 3f2a...                          Thu 2026-09-04 09:02:36 EST Thu 2026-09-04 11:10:02 EST
#Storage=auto
```
Un solo arranque y sin carpeta en `/var/log`: volátil. Si la carpeta ya existe y hay más de un arranque, ya es persistente: hacer la Parte 2 igual (no rompe nada).

---

## Parte 2 — Hacerlo persistente

**¿Qué le falta a `Storage=auto` para guardar en disco?**

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
ls -l /var/log/journal/
journalctl --disk-usage
```

Foto.

**Comprobar:**
```
total 0
drwxr-sr-x+ 2 root systemd-journal 60 ... 8c1f0e2a5b3d4c6e9f0a1b2c3d4e5f60
Archived and active journals take up 8.0M in the file system.
```
Con `auto`, alcanza con que exista la carpeta. Adentro hay una subcarpeta con el identificador de la máquina. Después del reinicio de la tarea, `--list-boots` va a mostrar dos arranques.

---

## Parte 3 — Cómo rota RHEL sus logs

**¿Cada cuánto se cortan `messages` y `secure`, y cuántas copias quedan?**

```bash
grep -v '#' /etc/logrotate.conf
cat /etc/logrotate.d/rsyslog
```

Foto.

**Comprobar:**
```
weekly
rotate 4
create
dateext
include /etc/logrotate.d
/var/log/cron
/var/log/maillog
/var/log/messages
/var/log/secure
/var/log/spooler
{
    missingok
    sharedscripts
    postrotate
        /usr/bin/systemctl -s HUP kill rsyslog.service >/dev/null 2>&1 || true
    endscript
}
```
Semanal, 4 copias, archivo nuevo después de rotar, sufijo con fecha (`messages-20260901`). `postrotate` manda `HUP` a rsyslog para que abra el archivo nuevo.

---

## Parte 4 — Rotación para `monitor.log`, forzada

```bash
sudo vim /etc/logrotate.d/monitor
```

`i`, pegar, `Esc`, `:wq`:

```
/var/log/monitor.log {
    daily
    rotate 7
    dateext
    compress
    delaycompress
    missingok
    notifempty
    create 0600 root root
    postrotate
        /usr/bin/systemctl -s HUP kill rsyslog.service
    endscript
}
```

```bash
sudo logrotate -d /etc/logrotate.d/monitor
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor.log*
sleep 31
sudo tail -1 /var/log/monitor.log
```

Foto.

**Comprobar:**
```
considering log /var/log/monitor.log
  Now: 2026-09-04 11:15
  log does not need rotating ...
-rw-------. 1 root root    0 Sep  4 11:15 /var/log/monitor.log
-rw-------. 1 root root 1840 Sep  4 11:15 /var/log/monitor.log-20260904
Sep  4 11:15:40 rhel01 monitor[5310]:  11:15:40 up 2:13,  1 user,  load average: ...
```
`-d` solo explica. `-f` fuerza: el viejo queda con la fecha, el nuevo arranca vacío, y a los 30 segundos `monitor` ya escribe en el nuevo.
