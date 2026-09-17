# Día 4 — Procesos, servicios, logs y software

## Qué vamos a hacer

1. **Procesos** — ver qué corre, quién lo lanzó, cuánto consume, y terminar lo que sobra
2. **Servicios con systemd** — arrancar, detener y habilitar servicios; crear uno propio que se reinicia solo
3. **Logs** — `journalctl` y `/var/log`: encontrar el rastro de lo que pasó
4. **Software** — instalar con `dnf`, consultar con `rpm`, deshacer una instalación, y repositorios: EPEL y uno local desde la ISO

Ayer controlamos **quién** puede hacer qué. Hoy vemos **qué está pasando** en el servidor, **cómo queda registrado**, y **de dónde sale** el software que corre ahí.

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 10 | **Comandos:** procesos — PID, estados, señales, jobs, prioridad, carga (explicado en consola) |
| 1 | 15 | **Lab 1:** radiografía de procesos — guiado: todos tipean, después leemos la salida |
| 1 | 25 | **Lab 2:** proceso descontrolado — encontrarlo, terminarlo, pausar, `nohup`, `nice` — guiado, foto por parte |
| 2 | 10 | **Comandos:** systemd — unidades, activo vs habilitado, leer `status`, unit file (explicado en consola) |
| 2 | 25 | **Lab 1:** controlar servicios con `sshd` y `httpd` — yo hago el primero, ustedes los demás, foto |
| 2 | 20 | **Lab 2:** servicio propio `monitor.service` y hora del sistema — guiado, foto por parte |
| — | 15 | Descanso |
| 3 | 10 | **Comandos:** logs — `journald`, `rsyslog`, `/var/log`, `logrotate` (explicado en consola) |
| 3 | 12 | **Lab 1:** `logger`, regla propia, accesos fallidos, filtros de `journalctl` — guiado, foto por parte |
| 3 | 8 | **Lab 2:** journal persistente y rotación de `monitor.log` — guiado, foto por parte |
| 4 | 10 | **Comandos:** software — `rpm`, `dnf`, repositorios, firmas (explicado en consola) |
| 4 | 25 | **Lab 1:** consultar, instalar, quitar y deshacer — yo hago el primero, ustedes los demás, foto |
| 4 | 10 | **Lab 2:** repositorios Red Hat, CodeReady y EPEL — guiado, foto por parte |
| 4 | 10 | **Lab 3:** repositorio local desde la ISO — guiado, foto por parte |
| — | 0 | Reto individual — si no se llega en clase, es tarea |
| — | 10 | Cierre y snapshot |
| — | 25 | Colchón (margen para imprevistos) |

---

## Al terminar

Saber qué corre en el servidor y por qué, administrar servicios, leer los logs para diagnosticar, e instalar software incluso sin internet.

---

## Tarea

1. **Snapshot `dia04-fin`** con la VM apagada. Antes de apagar: `sudo reboot`, y al volver `systemctl is-active monitor httpd` tiene que decir `active` dos veces y `journalctl --list-boots` tiene que mostrar dos arranques. **No borrar** `httpd`, `monitor.service`, EPEL ni el repo del DVD: se usan los días siguientes.
2. **Verificar la suscripción**, porque mañana se instala más software: `sudo dnf repolist` tiene que listar BaseOS y AppStream. Mañana también se agrega una segunda placa de red: tener el hipervisor (VirtualBox o UTM) a mano.
3. **El reto** (tres tickets) si no se hizo en clase: pegar en la terminal la línea que mando por el chat, resolver los tres, y mandar la salida de la verificación por el chat.
