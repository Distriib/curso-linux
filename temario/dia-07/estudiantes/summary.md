# Día 7 — Scripts en Bash y tareas programadas

## Qué vamos a hacer

1. **Primer script** — Un archivo que se ejecuta por nombre: `#!/bin/bash`, permiso `x`, `~/bin`, variables y argumentos
2. **Decisiones** — `if`, pruebas sobre archivos, códigos de salida, `bash -x` y `logger`
3. **Bucles** — `for` y `while read` sobre listas, archivos y un log; `backup.sh` con retención
4. **Scripts de administración** — altas de usuarios en lote y alerta de uso de disco
5. **Programar tareas** — `cron`, `at` y temporizadores de systemd

Hasta hoy cada comando se tipeaba a mano, uno por uno. Hoy se dejan escritos en un archivo, y el servidor los ejecuta solo a la hora que le digamos.

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 10 | **Comandos:** anatomía de un script, variables y argumentos (explicado en consola) |
| 1 | 20 | **Lab:** primer script — yo hago el primero, ustedes los demás, foto |
| 2 | 10 | **Comandos:** `if`, pruebas, `exit`, `bash -x`, `logger` (explicado en consola) |
| 2 | 20 | **Lab:** un script que decide — yo hago el primero, ustedes los demás, foto |
| 3 | 10 | **Comandos:** `for`, `while read`, funciones (explicado en consola) |
| 3 | 15 | **Lab 1:** bucles sobre un log — yo hago el primero, ustedes los demás, foto |
| 3 | 15 | **Lab 2:** `backup.sh` con retención — yo hago el primero, ustedes los demás, foto |
| — | 15 | Descanso |
| 4 | 5 | **Comandos:** scripts de administración (explicado en consola) |
| 4 | 15 | **Lab 1:** altas de usuarios en lote — yo hago el primero, ustedes los demás, foto |
| 4 | 15 | **Lab 2:** alerta de disco — yo hago el primero, ustedes los demás, foto |
| 5 | 15 | **Comandos:** `cron`, `at` y temporizadores de systemd (explicado en consola) |
| 5 | 20 | **Lab 1:** `cron` — yo hago el primero, ustedes los demás, foto |
| 5 | 10 | **Lab 2:** `at` — yo hago el primero, ustedes los demás, foto |
| 5 | 15 | **Lab 3:** temporizador de systemd — yo hago el primero, ustedes los demás, foto |
| — | 0 | Reto individual — solo si sobra tiempo; si no, es tarea |
| — | 10 | Cierre y snapshot |
| — | 20 | Colchón (margen para imprevistos) |

---

## Al terminar

Escribir un script que valida, decide y deja rastro, y dejarlo corriendo solo todos los días con `cron` o con un temporizador de systemd.

---

## Tarea

1. **Snapshot `dia07-fin`** con la VM apagada. Antes de apagar, no borrar nada: `crond`, `atd` y `monitor-disco.timer` quedan activos, y los scripts quedan en `~/bin` y `/usr/local/bin`.
2. **Verificar la suscripción**, porque mañana se instala software: `sudo dnf repolist` tiene que listar BaseOS y AppStream. Si dice `This system is not registered`: `sudo subscription-manager register`.
3. **El reto** (Ticket PGN-1187) si no se hizo en clase. Mandar la salida de la verificación por el chat.
