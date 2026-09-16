# Día 3 — Usuarios, grupos, sudo y permisos

## Qué vamos a hacer

1. **Quién es quién** — Usuarios, grupos, UID/GID, y dónde los guarda el sistema
2. **Crear y administrar cuentas** — Altas, contraseñas, caducidad y bloqueo
3. **Delegar privilegios** — `sudo` con reglas limitadas y auditables
4. **Permisos y carpetas compartidas** — `rwx`, octal, `umask`, setgid, sticky bit y ACL

Hasta hoy la pregunta era "**cómo** hago X". Hoy cambia a "**quién** puede hacer X, y cómo lo controlo".

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 15 | **Concepto:** el modelo de usuarios — UID, `/etc/passwd`, `/etc/shadow`, grupos (explicado en consola) |
| 1 | 20 | **Lab 1.1:** radiografía de las cuentas — guiado: todos tipean, después leemos la salida |
| 2 | 15 | **Comandos:** usuarios y grupos — se leen rápido, después el lab |
| 2 | 25 | **Lab:** crear usuarios y grupos — yo hago el primero, ustedes los demás, foto |
| 2 | 10 | **Lab:** política de contraseñas y bloqueo — si hay tiempo |
| — | 15 | Descanso |
| 3 | 10 | **Concepto:** `su`, `sudo` y `sudoers` (explicado en consola) |
| 3 | 20 | **Lab 3.1:** delegar con `sudo` — guiado: todos tipean, después leemos la salida |
| 4 | 20 | **Concepto:** permisos, `umask`, bits especiales, ACL (explicado en consola) |
| 4 | 15 | **Lab 4.1:** permisos y `umask` — guiado: todos tipean, después leemos la salida |
| 4 | 25 | **Lab 4.2:** carpetas compartidas en `/srv` — guiado: todos tipean, después leemos la salida |
| — | 20 | Reto individual — solo si sobra tiempo; si no, es tarea |
| — | 10 | Cierre y snapshot |
| — | 20 | Colchón (margen para imprevistos) |

---

## Al terminar

Crear usuarios con política de contraseñas, dar permisos de administrador acotados a comandos concretos, y montar carpetas compartidas donde cada área ve solo lo que le corresponde.

---

## Tarea

1. **Snapshot `dia03-fin`** con la VM apagada. **No borrar** los usuarios, grupos ni `/srv/*`: se reutilizan el Día 10.
2. **Verificar la suscripción**, porque mañana se instala software: `sudo dnf repolist` tiene que listar BaseOS y AppStream. Si dice `This system is not registered`: `sudo subscription-manager register`.
3. **El reto** (Ticket #2026-0312) si no se hizo en clase. Mandar la salida de la verificación por el chat.
