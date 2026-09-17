# Comandos — logs

## Dos sistemas que conviven

```
servicio ──▶ journald ──▶ journalctl          (binario, filtrable, puede borrarse al reiniciar)
                │
                └──────▶ rsyslog ──▶ /var/log/messages, /var/log/secure ...   (texto, permanente)
```

- **journald** recibe todo lo que escriben los servicios y le agrega quién, cuándo, qué unidad. Se consulta con `journalctl`.
- **rsyslog** lee del journal y escribe los archivos de texto de `/var/log` según reglas. Se leen con `tail`, `grep`, `less`.

```bash
sudo tail -3 /var/log/messages
sudo tail -3 /var/log/secure
journalctl -n 3 --no-pager
```

## `/var/log` — los archivos que hay que conocer

| Archivo | Qué tiene |
|---|---|
| `messages` | todo lo general. El primero que se abre |
| `secure` | autenticación: `sshd`, `sudo`, `su`, `passwd` |
| `cron` | tareas programadas |
| `boot.log` | mensajes del arranque |
| `dnf.log` | qué se instaló y cuándo |
| `audit/audit.log` | auditoría del kernel y SELinux (Día 8) |
| `wtmp`, `btmp`, `lastlog` | binarios: los lee `last`, `lastb`, `lastlog` |
| `httpd/access_log`, `httpd/error_log` | Apache |

## Regla de rsyslog: `facilidad.prioridad   destino`

| Facilidad = **quién** | Prioridad = **cuán grave** (de menor a mayor) |
|---|---|
| `auth`, `authpriv`, `cron`, `daemon`, `kern`, `mail`, `user`, `local0`…`local7` (para uso propio) | `debug`, `info`, `notice`, `warning`, `err`, `crit`, `alert`, `emerg` |

```
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                  /var/log/secure
local0.*                                    /var/log/monitor.log
& stop
```

- `*.info` = "info **o más grave**". `mail.none` = excluir.
- `& stop` = "lo que coincidió con la regla de arriba, no sigas procesándolo".
- Las de RHEL están en `/etc/rsyslog.conf`; las propias van en `/etc/rsyslog.d/nombre.conf`.
- `sudo rsyslogd -N1` valida la sintaxis antes de reiniciar. Después, `sudo systemctl restart rsyslog`.

```bash
logger -p local0.notice -t prueba "hola desde logger"
```

`logger` escribe un mensaje en el log con la facilidad, prioridad (`-p`) y etiqueta (`-t`) que uno quiera. Sirve para probar reglas y para que los scripts dejen rastro.

## `journalctl` — los filtros del día a día

| Filtro | Qué hace |
|---|---|
| `-u sshd` | solo esa unidad |
| `-f` | seguir en vivo (Ctrl+C sale) |
| `-n 5` | las últimas 5 líneas |
| `--no-pager` | sin visor, para pegar en el chat |
| `-b` | desde este arranque |
| `-b -1` | el arranque anterior: por qué se reinició |
| `-p err` | prioridad `err` o peor |
| `--since "1 hour ago"` · `--since today` · `--since "2026-09-04 09:00"` | por tiempo |
| `-t sudo` | por etiqueta |
| `--disk-usage` · `--list-boots` | cuánto ocupa · qué arranques tiene guardados |

```bash
journalctl -u sshd -n 3 --no-pager
sudo journalctl -p err -b --no-pager | tail -5
```

## El journal puede ser volátil

`Storage=auto` en `/etc/systemd/journald.conf`: si existe `/var/log/journal/`, escribe ahí y **sobrevive** al reinicio; si no, escribe en `/run/log/journal/` (memoria) y **se pierde**. Hay que comprobarlo.

## `logrotate` — que los logs no llenen el disco

Rota (renombra con fecha, comprime, borra los viejos) según `/etc/logrotate.conf` y un archivo por servicio en `/etc/logrotate.d/`. Lo dispara `logrotate.timer` una vez al día.

| Comando | Qué hace |
|---|---|
| `sudo logrotate -d /etc/logrotate.d/monitor` | simular: explica y no toca nada |
| `sudo logrotate -f /etc/logrotate.d/monitor` | forzar la rotación ahora |
