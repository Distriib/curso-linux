# Día 10 — Diagnóstico, recuperación y reto final

## Qué vamos a hacer

1. **Método de diagnóstico** — Convertir "la web no carga" en una causa, revisando el servidor capa por capa
2. **Logs a fondo** — `journalctl` por campos, `rsyslog` como colector de red, `logrotate` y `auditd`
3. **Recuperar el arranque** — Contraseña de root perdida y un `/etc/fstab` roto, desde la consola
4. **Construir `web01`** — Dejar el servidor con lo que armamos en el curso y tomar un snapshot
5. **Reto integrador** — Cinco tickets sobre el servidor averiado, sin pistas

Hasta hoy armamos cosas que funcionan. Hoy nos entregan cosas rotas: el trabajo es encontrar **qué** se rompió y dejarlo arreglado **de forma que aguante un reinicio**.

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 10 | **Comandos:** el método en 7 pasos y las capas, con el primer comando de cada una (explicado en consola) |
| 1 | 18 | **Lab 1.1:** radiografía del servidor — guiado: todos tipean, después leemos la salida |
| 2 | 8 | **Comandos:** journald, rsyslog, auditd y logrotate: quién guarda qué (explicado en consola) |
| 2 | 10 | **Lab 2.1:** journald por arranques y por campos — guiado: todos tipean, después leemos la salida |
| 2 | 12 | **Lab 2.2:** rsyslog: regla propia y colector por red — yo hago el primero, ustedes los demás, foto |
| 2 | 8 | **Lab 2.3:** logrotate — yo hago el primero, ustedes los demás, foto |
| 2 | 10 | **Lab 2.4:** auditoría: quién tocó qué y quién intentó entrar — guiado: todos tipean, después leemos la salida |
| 3 | 10 | **Comandos:** cómo arranca RHEL, GRUB, targets y `rd.break` (explicado en consola) |
| 3 | 8 | **Lab 3.1:** GRUB visible y kernels — guiado: todos tipean, después leemos la salida |
| 3 | 15 | **Lab 3.2:** contraseña de root perdida — guiado, en la consola de la VM |
| — | 15 | Descanso |
| 3 | 12 | **Lab 3.3:** el servidor no arranca: reparar `/etc/fstab` — guiado, en la consola de la VM |
| 4 | 5 | **Comandos:** qué tiene `web01` y cómo se verifica cada pieza (explicado en consola) |
| 4 | 20 | **Lab 4.1:** construir `web01` y snapshot `dia10-pre-romper` — yo hago el primero, ustedes los demás, foto |
| 5 | 50 | **Reto integrador:** cinco tickets sobre el servidor averiado — solos, sin pistas; lo que no se termina queda de tarea |
| — | 10 | Cierre y snapshot |
| — | 19 | Colchón (margen para imprevistos) |

---

## Al terminar

Diagnosticar un servidor con fallas siguiendo un método, leer sus logs para encontrar la causa, y recuperar uno que ni siquiera arranca.

---

## Tarea

1. **Snapshot `dia10-fin`** con la VM apagada, después de que la verificación del reto dé todo bien. **Conservar** también `dia10-pre-romper`: sirve para repetir el reto.
2. **Repetir el reto** en la semana: restaurar `dia10-pre-romper`, correr `romper.sh` otra vez y resolver los cinco tickets cronometrando. Meta: los tres primeros en menos de 20 minutos.
3. **Los tickets que quedaron sin resolver** en clase. Mandar por el chat la salida de la verificación y las cuatro líneas de cada ticket.
