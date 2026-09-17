# 2 — Logs: quién guarda qué

## Cuatro programas, cuatro trabajos

| Programa | Qué hace | Dónde guarda | Se consulta con |
|---|---|---|---|
| **journald** | Recoge **todo**: servicios, kernel, syslog, auditoría. Binario, indexado por campos | `/var/log/journal/` (persistente) | `journalctl` |
| **rsyslog** | Lee del journal y escribe archivos de texto según reglas. Envía y recibe por red | `/var/log/messages`, `secure`, `cron`, `maillog` | `tail`, `grep` |
| **auditd** | Vigila los archivos y acciones que se le indican. Registra las denegaciones de SELinux | `/var/log/audit/audit.log` | `ausearch`, `aureport` |
| **logrotate** | Rota, comprime y borra logs viejos para que no llenen el disco | `/etc/logrotate.conf`, `/etc/logrotate.d/` | `logrotate -d` / `-f` |

journald es la **grabadora** de todo el edificio; rsyslog el **archivista** que copia lo importante a carpetas y lo manda a la central; auditd la **cámara** que graba solo las puertas que se le indican; logrotate el que **vacía el archivero** cada noche.

```bash
journalctl --list-boots
ls /var/log/journal/
ls /var/log/
systemctl is-active rsyslog auditd
systemctl list-timers logrotate.timer --no-pager
```

## `journalctl` — filtrar por campo

Cada mensaje trae campos. `-o verbose` los muestra todos, y cualquiera sirve para filtrar.

| Filtro | Qué trae |
|---|---|
| `-b` / `-b -1` | este arranque / el anterior |
| `--list-boots` | los arranques guardados |
| `-p err` | prioridad `err` o peor |
| `-u httpd` | la unidad `httpd`, más lo que systemd dice **sobre** ella |
| `_SYSTEMD_UNIT=httpd.service` | solo lo que escribió la unidad |
| `_COMM=sudo` | por nombre de programa |
| `_UID=1000` | por usuario |
| `_PID=1` | por proceso |
| `-k` | solo kernel: como `dmesg`, pero con fecha y de arranques anteriores |
| `-t backup` | por etiqueta (lo que `logger -t` pone) |
| `-o verbose` / `-o json-pretty` / `-o short-iso` | todos los campos / en JSON / fecha ISO |
| `--disk-usage` / `--vacuum-time=2weeks` | cuánto ocupa / borrar los archivados de más de dos semanas |

Límite y persistencia: un archivo en `/etc/systemd/journald.conf.d/` con `Storage=persistent` y `SystemMaxUse=500M`.

## rsyslog — anatomía de una regla

```
local5.err      /var/log/messages
  │     │             └ destino: un archivo · @IP:514 (UDP) · @@IP:514 (TCP)
  │     └ prioridad: esta y las más graves
  └ facilidad: quién habla
```

Facilidades: `auth`, `authpriv`, `cron`, `kern`, `mail`, `daemon`, `user`, `local0`…`local7` (para uso propio).
Prioridades, de menor a mayor: `debug` < `info` < `notice` < `warning` < `err` < `crit` < `alert` < `emerg`.

| Escritura | Significa |
|---|---|
| `mail.warning` | `warning` y más graves |
| `mail.=warning` | solo `warning` |
| `mail.none` | nada de `mail` |
| `local5.*  stop` | lo que sea `local5` **no sigue** a las reglas de abajo (evita duplicados en `messages`) |

Las reglas propias van en `/etc/rsyslog.d/NOMBRE.conf` y se leen **antes** que las de `/etc/rsyslog.conf`. Siempre `sudo rsyslogd -N1` antes de reiniciar.

## Recibir logs de otros servidores (colector)

```
module(load="imtcp")
ruleset(name="desde_remotos") {
    action(type="omfile" file="/var/log/remoto.log")
}
input(type="imtcp" port="514" ruleset="desde_remotos")
```

Lo que entra por el 514 va a su propio archivo y **no** pasa por las reglas normales. Sin el `ruleset`, un servidor que también envía se mandaría sus propios mensajes a sí mismo para siempre.

| Paso | Comando |
|---|---|
| SELinux: 514/tcp no es de syslog (es de `rsh`) | `sudo semanage port -m -t syslogd_port_t -p tcp 514` |
| Firewall, las dos zonas | `sudo firewall-cmd --permanent --add-port=514/tcp` · `--zone=internal` · `--reload` |
| Probar sin otro servidor | `logger -n 127.0.0.1 -P 514 -T -t prueba "mensaje"` |
| Que un servidor envíe todo | en el cliente: `*.*  @@IP:514` |

## logrotate

```
/var/log/monitor-disco.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
}
```

| Opción | Significa |
|---|---|
| `daily` / `rotate 7` | rotar cada día, conservar 7 copias |
| `compress` / `delaycompress` | comprimir las viejas, menos la última rotada |
| `missingok` / `notifempty` | si no existe no es error / no rotar vacíos |
| `create 0640 root root` | el archivo nuevo nace con esos permisos |
| `postrotate` … `endscript` | comando a ejecutar después de rotar (avisar a un servicio) |

`sudo logrotate -d archivo` simula y explica. `-f` fuerza. El estado va en `/var/lib/logrotate/logrotate.status`. Lo dispara `logrotate.timer` cada día.

## auditd — anatomía de una regla

```
-w /etc/passwd -p wa -k passwd_changes
 │      │        │      └ llave: etiqueta para buscarlos después
 │      │        └ qué vigilar: w escritura · a atributos · r lectura · x ejecución
 │      └ el archivo o carpeta
 └ watch: vigilar
```

Van en `/etc/audit/rules.d/NOMBRE.rules`; `sudo augenrules --load` las carga; `sudo auditctl -l` las muestra. Sobreviven al reinicio.

| Comando | Qué hace |
|---|---|
| `sudo ausearch -k passwd_changes -ts recent -i` | eventos con esa llave, últimos 10 minutos, con nombres en vez de números |
| `sudo ausearch -m USER_CMD -ts today -i` | comandos ejecutados con `sudo` hoy |
| `sudo aureport --summary` | resumen: logins, fallos, cambios de cuentas |

`-ts recent` = últimos 10 minutos · `-ts today` · `-ts 08:00`.
