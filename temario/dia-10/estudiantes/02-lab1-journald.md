# Lab 2.1 — journald: arranques, campos y tamaño

Todos corren cada comando; después lo leemos juntos. Objetivo: mirar arranques anteriores, filtrar por campo, ver todos los campos de un mensaje y limitar el tamaño del journal.

---

## Parte 1 — ¿Cómo se apagó el servidor la última vez?

**¿El journal recuerda arranques anteriores? ¿Qué pasó al final del anterior?**

```bash
journalctl --list-boots | tail -3
journalctl -b -1 -p err --no-pager | tail -3
journalctl -b -1 -n 3 --no-pager
```

**Comprobar:**
```
-2 ...  ... 2026-09-... —... 2026-09-...
-1 ...  ... 2026-09-... —... 2026-09-...
 0 ...  ... 2026-09-16 ...
... rhel01 systemd[1]: ...    (las tres últimas líneas del arranque anterior)
```
`0` es este arranque, `-1` el anterior. Si las últimas líneas de `-1` dicen `Reached target ... Power-Off` o `Reboot`, se apagó bien; si terminan en cualquier otra cosa, se cortó. Si `--list-boots` muestra solo `0`, el journal no es persistente.

---

## Parte 2 — Un mensaje de prueba y los filtros

**¿Puedo buscar por programa, por usuario, por unidad?**

```bash
logger -t backup "OK: creado /backups/empresa-prueba.tar.gz (4.0K)"
journalctl -t backup -n 2 --no-pager
journalctl -u httpd -b --no-pager | tail -2
journalctl _SYSTEMD_UNIT=httpd.service -b --no-pager | tail -2
journalctl _COMM=sudo -n 3 --no-pager
journalctl _UID=1000 -n 2 --no-pager
journalctl -k -n 2 --no-pager
```

**Comprobar:**
```
... rhel01 backup[NNNN]: OK: creado /backups/empresa-prueba.tar.gz (4.0K)
... rhel01 systemd[1]: Started The Apache HTTP Server.
... rhel01 httpd[NNNN]: Server configured, listening on: port 80
... rhel01 sudo[NNNN]:  student : TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=...
... rhel01 kernel: ...
```
`-u httpd` muestra más que `_SYSTEMD_UNIT=`: agrega lo que systemd dice **sobre** la unidad (`Started ...`). `-k` es `dmesg` con fecha.

---

## Parte 3 — Todos los campos de un mensaje

**¿Qué campos tiene un mensaje, y cuáles puedo usar para filtrar?**

```bash
journalctl -t backup -n 1 -o verbose --no-pager
journalctl -t backup -n 1 -o json-pretty --no-pager | grep MESSAGE
journalctl -p warning -b -o short-iso --no-pager | tail -2
```

**Comprobar** (entre las líneas de `-o verbose`):
```
    PRIORITY=5
    SYSLOG_FACILITY=1
    SYSLOG_IDENTIFIER=backup
    MESSAGE=OK: creado /backups/empresa-prueba.tar.gz (4.0K)
    _UID=1000
    _COMM=logger
    _EXE=/usr/bin/logger
    _HOSTNAME=rhel01
        "MESSAGE" : "OK: creado /backups/empresa-prueba.tar.gz (4.0K)",
2026-09-16T...-0500 rhel01 ...
```
Cualquier línea `CAMPO=valor` sirve tal cual como filtro: `journalctl _EXE=/usr/bin/logger`. `PRIORITY=5` y `SYSLOG_FACILITY=1` son `user.notice`, lo que `logger` pone si no se le dice otra cosa.

---

## Parte 4 — Cuánto ocupa y cómo está configurado

**¿El journal puede llenar el disco?**

```bash
journalctl --disk-usage
ls /var/log/journal/
sudo journalctl --vacuum-time=2weeks
grep Storage /etc/systemd/journald.conf
grep SystemMaxUse /etc/systemd/journald.conf
```

**Comprobar:**
```
Archived and active journals take up ...M in the file system.
3a7c...                      (una carpeta con el ID de la máquina)
Vacuuming done, freed 0B of archived journals from /var/log/journal/...
#Storage=auto
#SystemMaxUse=
```
Los valores con `#` son los de fábrica: `auto` = persistente porque existe `/var/log/journal`; sin límite propio usa hasta el 10 % del disco. `--vacuum-time` borra solo archivos **archivados**, nunca el activo.

---

## Parte 5 — Fijar el límite con un archivo propio

**¿Cómo cambio la configuración sin editar el archivo del sistema?**

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo vim /etc/systemd/journald.conf.d/pgn.conf
```
Contenido:
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

**Comprobar:** `Archived and active journals take up ...` sin errores, y el archivo con las cuatro líneas. `Storage=persistent` obliga a guardar en disco aunque la carpeta no existiera: es la receta del examen.
