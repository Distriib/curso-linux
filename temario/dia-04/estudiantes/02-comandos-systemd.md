# Comandos — servicios con systemd

## systemd es PID 1 y maneja todo lo que arranca después

Habla en **unidades**: archivos de texto cuya extensión dice qué son.

| Tipo | Ejemplo | Para qué |
|---|---|---|
| `.service` | `sshd.service`, `httpd.service` | un servicio (un programa que corre de fondo) |
| `.target` | `multi-user.target`, `graphical.target` | un grupo de unidades: el "modo" en que arranca el sistema |
| `.timer` | `logrotate.timer` | ejecutar un `.service` según calendario |
| `.socket` | `cockpit.socket` | escuchar en un puerto y arrancar el servicio solo cuando alguien se conecta |
| `.mount` | `boot.mount` | un punto de montaje |

Dónde viven:

| Carpeta | Qué hay |
|---|---|
| `/usr/lib/systemd/system/` | las que instalan los paquetes. **No se editan** |
| `/etc/systemd/system/` | las del administrador. **Ganan** si hay una con el mismo nombre |

## Las dos preguntas — son independientes

| Pregunta | Cambiar | Consultar |
|---|---|---|
| ¿Está corriendo **ahora**? | `start` · `stop` · `restart` · `reload` | `is-active`, línea `Active:` |
| ¿Arranca **con el sistema**? | `enable` · `disable` | `is-enabled`, línea `Loaded:` |

- `enable --now` = las dos cosas juntas.
- Activo y deshabilitado = funciona hasta el próximo reinicio ("ayer andaba y hoy no").
- `enable` solo crea un enlace en `/etc/systemd/system/multi-user.target.wants/`. `disable` lo borra.
- `mask` = prohibir: enlace a `/dev/null`; nadie puede arrancarlo, ni a mano ni como dependencia. `unmask` deshace.
- `reload` = releer configuración sin cortar conexiones (manda `HUP`). `restart` = parar y arrancar: cambia el PID y corta clientes.

```bash
systemctl status sshd
systemctl is-active sshd
systemctl is-enabled sshd
```

## Leer `status` completo

```
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Thu 2026-09-04 10:12:01 EST; 5s ago
   Main PID: 4321 (httpd)
      Tasks: 177 (limit: 23000)
     Memory: 24.1M
     CGroup: /system.slice/httpd.service
             ├─4321 /usr/sbin/httpd -DFOREGROUND
   ... últimas líneas del log de la unidad ...
```

- `Loaded:` de qué archivo viene · si está `enabled`/`disabled`/`masked` · qué haría RHEL por defecto (`preset`).
- `Active:` puede ser `active (running)`, `inactive (dead)`, `failed`, `activating`.
- `CGroup:` **todos** los procesos que systemd considera parte del servicio. `stop` los mata a todos.
- Abajo, el log: la mitad de los diagnósticos terminan ahí.

## `systemctl` — lo que se usa

| Comando | Qué hace |
|---|---|
| `systemctl list-units --type=service --state=running` | qué servicios corren ahora |
| `systemctl list-unit-files --type=service --state=enabled` | cuáles arrancan con el sistema |
| `systemctl --failed` | cuáles fallaron (lo deseable: ninguno) |
| `systemctl cat sshd` | el archivo de la unidad, tal cual |
| `systemctl show -p MainPID --value httpd` | una propiedad, sin adornos |
| `systemctl list-timers` | temporizadores y cuándo corren |
| `systemctl get-default` / `set-default multi-user.target` | con qué target arranca |
| `sudo systemctl daemon-reload` | releer los archivos de unidad. **Siempre** después de crear o editar uno |

Si una salida te deja en un visor, `q` para salir; `--no-pager` lo evita.

## Anatomía de un unit file

```
[Unit]                         qué es y con qué se relaciona
Description=Monitor de carga
After=network.target           arrancar después de la red

[Service]                      cómo se ejecuta
Type=simple                    el proceso de ExecStart ES el servicio
ExecStart=/usr/local/bin/monitor.sh    ruta completa, obligatoria
Restart=on-failure             si termina mal, relanzarlo
RestartSec=5                   esperando 5 segundos

[Install]                      qué hace enable
WantedBy=multi-user.target     enable crea el enlace en multi-user.target.wants/
```

Después de escribirlo: `sudo systemd-analyze verify /etc/systemd/system/monitor.service` (no imprime nada si está bien) y `sudo systemctl daemon-reload`.

## La hora

Los logs se ordenan por hora: con la hora mal no se puede investigar nada.

```bash
timedatectl
chronyc sources
```

`chronyd` sincroniza la hora por red (NTP). `timedatectl` muestra zona horaria y si está sincronizado. Panamá: `America/Panama`.
