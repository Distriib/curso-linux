# Comandos — usuarios y grupos (guía del instructor)

Se lee rápido, de arriba a abajo, y se pasa al lab. No demostrar acá: todo se hace en el lab con ellos.

**Qué es `useradd`:** crea la cuenta. Un solo comando toca cinco lugares: `passwd`, `shadow`, `group` (el grupo privado), la carpeta home (copiada de la plantilla `/etc/skel`), y el buzón en `/var/spool/mail`.
**Qué es `-u` / `-g`:** el número. Sin la opción, el sistema elige el próximo libre (1001, 1002…). Con la opción, elegís vos. En clase lo ponemos para que a todos les salga igual; en la vida real, para que la misma persona tenga el mismo número en todos los servidores (los permisos se guardan por número, no por nombre). Los números altos (2001, 3001) son para no chocar con los que el sistema asigna solo.
**Qué es `-c`:** un comentario libre. Va al campo 5 de `passwd`. Nombre completo y área, por convención.
**Qué es `-m`:** crear la carpeta home. En RHEL se crea igual sin `-m`, pero se escribe por costumbre (en otras distros es obligatorio).
**Qué es `usermod`:** modificar una cuenta que ya existe. `-aG` es la opción del día.
**Qué es `--stdin`:** `passwd` normalmente pregunta la contraseña. Con `--stdin` la lee de la tubería (`echo 'clave' | sudo passwd --stdin usuario`). Sirve para no escribirla a mano cuatro veces.
**Qué es `passwd -S`:** el estado de la contraseña sin leer `shadow`: `LK` bloqueada o sin poner, `PS` puesta, `NP` sin contraseña.
**Qué es `su - ana`:** abrir una sesión como ana. Pide la contraseña **de ana**, no la tuya. El guion importa: sin él, cambiás de nombre pero seguís parado en tu carpeta con tu entorno.

---

## Lo que hay que decir sí o sí

**El `-a` de `usermod -aG`.** Despacio: *"`usermod -G sistemas ana` no agrega: **reemplaza** toda su lista de grupos. Si ana estaba en `wheel` y hacen eso, ana **sale** de `wheel`. Para agregar es `-aG`, con la `a` de *append*."* Es el error #1 del tema y en un servidor real deja a alguien sin acceso a todo lo demás.

**Los grupos se leen al iniciar sesión.** Si agregás a alguien a un grupo mientras tiene la sesión abierta, `id` lo muestra pero no le funciona hasta que salga y vuelva a entrar. Es el ticket "me agregaste y sigo sin poder".

**Root puede poner contraseñas débiles.** `passwd` avisa `BAD PASSWORD` y la acepta igual. Un usuario cambiando la suya, no. "La política se le aplica al usuario, no al administrador."

**Contraseña del curso: `Pgn.2026`.** Decirla antes del lab para que nadie invente.

## Plazos y bloqueo

Solo si se llega al Lab 2 (política de contraseñas). Ahí:
- `chage -M 90 -m 1 -W 7`: la política **es** ponerle esos tres números a cada usuario. No hay que "crearla" antes. `-M 90` = la contraseña dura 90 días; `-W 7` = aviso 7 días antes; `-m 1` = no puede cambiarla dos veces el mismo día (para que no vuelva a la anterior).
- Tres bloqueos que no son lo mismo: `passwd -l` (solo la contraseña; una llave SSH sigue entrando), `nologin` (sin shell; otros servicios siguen), `chage -E 0` (la cuenta entera; nada entra).
- `userdel -r`: borra la carpeta también. Sin `-r`, queda la carpeta huérfana.
