# Comandos — programar tareas

## Tres herramientas, tres preguntas

| ¿La tarea…? | Herramienta | Quién la define |
|---|---|---|
| se repite (todos los días, cada 15 min) | `cron` | cada usuario con `crontab`; root en `/etc/cron.d/` |
| corre una sola vez, más tarde | `at` | cada usuario |
| se repite y tiene que integrarse con systemd (journal, recuperar lo perdido) | temporizador de systemd | root en `/etc/systemd/system/` |

## `cron`: anatomía de una línea

```
30   23   *   *   1-5   /home/student/bin/backup.sh >> /home/student/backup.log 2>&1
│    │    │   │   │     └ el comando, con rutas completas y la salida a un archivo
│    │    │   │   └ día de la semana (0-7; 0 y 7 = domingo)
│    │    │   └ mes (1-12)
│    │    └ día del mes (1-31)
│    └ hora (0-23)
└ minuto (0-59)
```

| Campos | Cuándo |
|---|---|
| `* * * * *` | cada minuto |
| `*/15 * * * *` | cada 15 minutos |
| `30 23 * * *` | todos los días a las 23:30 |
| `0 2 * * 1-5` | lunes a viernes a las 02:00 |
| `0 8,13 * * *` | a las 08:00 y a las 13:00 |
| `15 3 1 * *` | el día 1 de cada mes a las 03:15 |
| `@reboot` | al arrancar |

`*` = todos · `,` = lista · `-` = rango · `*/N` = cada N.

## El crontab del usuario

| Comando | Qué hace |
|---|---|
| `crontab -l` | ver el mío |
| `crontab ARCHIVO` | instalar ese archivo como mi crontab (reemplaza el anterior) |
| `crontab -e` | editarlo en el editor |
| `crontab -r` | **borrarlo sin preguntar** (está al lado de la `e`) |
| `sudo crontab -u ana -l` | ver el de otro usuario |

Arriba del crontab, tres líneas que evitan tickets:

```
SHELL=/bin/bash
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""
```

`cron` arranca con un `PATH` mínimo: un script que funciona a mano falla en `cron` con `command not found`. La salida de cada tarea se manda por correo, y este servidor no tiene correo: se pierde. Por eso siempre `>> /ruta/log 2>&1`.

## `cron` del sistema

```bash
cat /etc/crontab
cat /etc/cron.d/0hourly
ls /etc/cron.hourly /etc/cron.daily
sudo tail -3 /var/log/cron
```

| Lugar | Qué hay |
|---|---|
| `/etc/crontab`, `/etc/cron.d/*` | líneas con un **sexto campo: el usuario** que ejecuta |
| `/etc/cron.hourly/`, `cron.daily/`, `cron.weekly/`, `cron.monthly/` | scripts sueltos; los ejecuta `run-parts` |
| `/etc/cron.deny` | usuarios que **no** pueden usar `crontab` (vacío por defecto) |
| `/var/log/cron` | qué ejecutó `cron` y cuándo: `(student) CMD (comando)` |

## `at`: una sola vez

| Comando | Qué hace |
|---|---|
| `at now + 2 minutes` | abre el prompt `at>`; se escriben los comandos y se cierra con `Ctrl+D` |
| `echo "comando" \| at 17:30` | lo mismo, en una línea |
| `atq` | ver los trabajos pendientes |
| `atrm 2` | borrar el trabajo 2 |

Necesita el paquete `at` y el servicio `atd` activo.

## Temporizador de systemd: dos archivos con el mismo nombre

```
/etc/systemd/system/monitor-disco.service   ← QUÉ se ejecuta
[Service]
Type=oneshot                                ← corre y termina
ExecStart=/usr/local/bin/monitor-disco.sh 80

/etc/systemd/system/monitor-disco.timer     ← CUÁNDO
[Timer]
OnCalendar=*:0/10                           ← cada 10 minutos
Persistent=true                             ← si el servidor estaba apagado, lo corre al arrancar
[Install]
WantedBy=timers.target
```

Se habilita **el timer**, no el service: `sudo systemctl enable --now monitor-disco.timer`.

| `OnCalendar=` | Cuándo |
|---|---|
| `*:0/10` | cada 10 minutos |
| `hourly` · `daily` · `weekly` | cada hora · cada día a las 00:00 · lunes 00:00 |
| `*-*-* 02:00` | todos los días a las 02:00 |
| `Mon..Fri 02:00` | lunes a viernes a las 02:00 |
| `Sat *-*-* 23:00` | sábados a las 23:00 |
| `*-*-01 03:00` | el día 1 de cada mes a las 03:00 |

| Comando | Qué hace |
|---|---|
| `systemctl list-timers` | todos los temporizadores: próxima y última ejecución |
| `systemd-analyze calendar "Mon..Fri 02:00"` | comprobar una expresión antes de usarla |
| `sudo journalctl -u monitor-disco.service` | lo que imprimió el script (systemd lo guarda solo) |

```bash
systemctl list-timers --no-pager | head -5
```

## Quién limpia `/tmp`

```bash
grep -v "#" /usr/lib/tmpfiles.d/tmp.conf
```

```
q /tmp 1777 root root 10d
│  │    │    │    │    └ borrar lo que tenga más de 10 días
│  │    │    │    └ grupo
│  │    │    └ dueño
│  │    └ permisos
│  └ ruta
└ tipo: crear la carpeta si no existe y limpiarla
```

Lo ejecuta `systemd-tmpfiles-clean.timer` una vez al día. Reglas propias: `/etc/tmpfiles.d/`.
