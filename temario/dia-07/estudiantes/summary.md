# Día 7 — Scripts en Bash y tareas programadas

## Qué vamos a hacer

1. **Primer script** — Un archivo que se ejecuta por nombre: `#!/bin/bash`, permiso `x`, `~/bin`, variables y argumentos
2. **Decisiones** — `if`, pruebas sobre archivos, códigos de salida, `bash -x` y `logger`
3. **Bucles** — `for` y `while read` sobre listas, archivos y un log; `backup.sh` con retención
4. **Scripts de administración** — altas de usuarios en lote y alerta de uso de disco
5. **Programar tareas** — `cron`, `at` y temporizadores de systemd

Hasta hoy cada comando se tipeaba a mano, uno por uno. Hoy se dejan escritos en un archivo, y el servidor los ejecuta solo a la hora que le digamos.

## Los laboratorios del día

| Lab | Qué se hace |
|---|---|
| 1.1 | Primer script: `hola.sh`, `args.sh` y `reporte.sh` en `~/bin` |
| 2.1 | `revisar.sh`: un script que decide y devuelve códigos de salida |
| 3.1 | Bucles `for` y `while read` sobre un log |
| 3.2 | `backup.sh`: comprimir con fecha y borrar los respaldos viejos |
| 4.1 | Altas de usuarios en lote desde un archivo de texto |
| 4.2 | `monitor-disco.sh`: alerta cuando un disco pasa el umbral |
| 5.1 | `cron`: la trampa del `PATH` y el crontab definitivo |
| 5.2 | `at`: una tarea que corre una sola vez |
| 5.3 | Temporizador de systemd para `monitor-disco.sh` |

Cada lab termina con una sección **Solución** con todos los comandos. Mirarla después de intentarlo.

**Reto del día:** Ticket PGN-1187 — `limpiar-logs.sh`, programado por `cron` y por un temporizador de systemd.

## Dónde queda cada cosa

| Carpeta | Qué se deja ahí |
|---|---|
| `~/bin` | `hola.sh`, `args.sh`, `reporte.sh`, `revisar.sh`, `backup.sh`, `crear-usuarios.sh` |
| `/usr/local/bin` | `monitor-disco.sh` (lo ejecuta systemd como root) |
| `/datos/backups` | los respaldos que genera `backup.sh` |
| `/etc/systemd/system` | `monitor-disco.service` y `monitor-disco.timer` |

---

## Al terminar

Escribir un script que valida, decide y deja rastro, y dejarlo corriendo solo todos los días con `cron` o con un temporizador de systemd.

---

## Tarea

1. **Snapshot `dia07-fin`** con la VM apagada. Antes de apagar, no borrar nada: `crond`, `atd` y `monitor-disco.timer` quedan activos, y los scripts quedan en `~/bin` y `/usr/local/bin`.
2. **Verificar la suscripción**, porque mañana se instala software: `sudo dnf repolist` tiene que listar BaseOS y AppStream. Si dice `This system is not registered`: `sudo subscription-manager register`.
3. **El reto** (Ticket PGN-1187) si no se hizo en clase. Mandar la salida de la verificación por el chat.
