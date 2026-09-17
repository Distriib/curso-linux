# Lab — journald: arranques, campos y tamaño (comandos)

Guiado: todos tipean, se lee la salida juntos.

## Parte 1
```bash
journalctl --list-boots | tail -3
journalctl -b -1 -p err --no-pager | tail -3
journalctl -b -1 -n 3 --no-pager
```
Si a alguien `--list-boots` le muestra solo `0`: `sudo mkdir -p /var/log/journal` y `sudo systemctl restart systemd-journald`; desde el próximo arranque guarda en disco. Hoy sigue con lo que tiene.

## Parte 2
```bash
logger -t backup "OK: creado /backups/empresa-prueba.tar.gz (4.0K)"
journalctl -t backup -n 2 --no-pager
journalctl -u httpd -b --no-pager | tail -2
journalctl _SYSTEMD_UNIT=httpd.service -b --no-pager | tail -2
journalctl _COMM=sudo -n 3 --no-pager
journalctl _UID=1000 -n 2 --no-pager
journalctl -k -n 2 --no-pager
```
Si `-u httpd` sale vacío, Apache no está corriendo en esa VM: `sudo systemctl enable --now httpd` y repetir. Lo necesitan igual en el Bloque 4.

## Parte 3
```bash
journalctl -t backup -n 1 -o verbose --no-pager
journalctl -t backup -n 1 -o json-pretty --no-pager | grep MESSAGE
journalctl -p warning -b -o short-iso --no-pager | tail -2
```
El orden de los campos en `-o verbose` varía: que busquen `_COMM=logger` y `MESSAGE=` con la vista, no por posición.

## Parte 4
```bash
journalctl --disk-usage
ls /var/log/journal/
sudo journalctl --vacuum-time=2weeks
grep Storage /etc/systemd/journald.conf
grep SystemMaxUse /etc/systemd/journald.conf
```

## Parte 5
```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo vim /etc/systemd/journald.conf.d/pgn.conf
```
```
[Journal]
Storage=persistent
SystemMaxUse=500M
MaxRetentionSec=1month
```
```bash
sudo systemctl restart systemd-journald
journalctl --disk-usage
cat /etc/systemd/journald.conf.d/pgn.conf
```
