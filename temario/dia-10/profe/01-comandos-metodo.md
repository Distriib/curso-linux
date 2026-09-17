# 1 — Método de diagnóstico (guía del instructor)

Diez minutos. Tiene que quedar una sola cosa: **hay un orden**, y el orden es de abajo hacia arriba (arranque → kernel → servicio → red → permisos → SELinux → disco → aplicación). La tabla del primer comando no se lee entera: se señala que existe y que hoy la van a usar en el reto. Tipeás los tres comandos del bloque bash mientras hablás.

**Qué es un ticket:** el pedido que llega a la mesa de ayuda, con las palabras del usuario ("la web no carga"). Hoy los tickets son el reto.
**Qué es un síntoma vs una causa:** el síntoma es lo que ve el usuario; la causa es lo que está mal en el servidor. Un ticket trae síntoma; el trabajo es llegar a la causa.
**Qué es una hipótesis:** una suposición que se puede comprobar con un comando ("el servicio está caído" → `systemctl status`). Si el comando no la confirma, se pasa a la siguiente; no se "arregla por si acaso".
**Qué es una capa:** cada nivel del que depende un servicio. Si falla una capa de abajo, todo lo de arriba falla también, por eso se revisan en orden.
**Qué es persistente:** que sobrevive a un reinicio. `systemctl start` no es persistente, `enable --now` sí (Día 4). `firewall-cmd --add-service` sin `--permanent` no es persistente (Día 8). `chcon` no es persistente, `restorecon` con su regla sí (Día 8). `ip addr add` no es persistente, `nmcli` sí (Día 5).
**Qué es `systemd-analyze`:** herramienta de systemd que dice cuánto tardó el arranque (`time`) y qué servicio tardó más (`blame`, "culpar").
**Qué es `dmesg`:** los mensajes del kernel (discos detectados, tarjetas de red, errores de hardware). `-T` pone fecha legible en vez de segundos desde el arranque.
**Qué es OOM:** *Out Of Memory*. Cuando se acaba la memoria, el kernel mata el proceso que más consume y lo anota en `dmesg` como `Out of memory: Killed process`. Es la explicación de "el servicio se murió solo".
**Qué es `ss -tlnp`:** lista los puertos TCP que están escuchando (`-l`), con números (`-n`) y el proceso dueño (`-p`, necesita `sudo`). Lo usaron el Día 5.
**Qué es una zona del firewall:** un conjunto de reglas que se aplica según por dónde entra el tráfico. En esta VM: `public` para las interfaces, `internal` para la red host-only `192.168.56.0/24` (Día 8). Lo que se publica va en **las dos**.
**Qué es `getfacl`:** muestra las ACL de un archivo (Día 3). **Qué es `sudo -l -U usuario`:** qué puede ejecutar ese usuario con `sudo` (Día 3).
**Qué es un AVC:** *Access Vector Cache*: así se llama cada denegación de SELinux en el log de auditoría. `ausearch -m AVC -ts recent` las muestra de los últimos 10 minutos (Día 8).
**Qué es `getenforce` / `ls -Z` / `semanage port -l`:** el modo de SELinux, el contexto de un archivo, y la lista de puertos que cada servicio tiene permitido usar (Día 8).
**Qué es un inodo (`df -i`):** cada archivo ocupa un inodo, aparte de su espacio. Un disco puede tener espacio libre y aun así estar "lleno" porque se acabaron los inodos (millones de archivos chicos).
**Qué es `lsof +L1`:** lista archivos abiertos con menos de un enlace, o sea **borrados pero todavía abiertos** por algún proceso. Es el comando del Lab 1.1, Parte 7.
**Qué es `findmnt --verify`:** revisa `/etc/fstab` y dice si hay líneas que no van a poder montarse (Día 6). Es la comprobación que evita el ticket del Lab 3.3.
**Qué es `uptime` / load average:** cuánto lleva encendido y la carga promedio en 1, 5 y 15 minutos. Carga = procesos esperando CPU o disco; sana si es menor que la cantidad de CPU.
**Qué es `free -m` / `vmstat`:** memoria libre y usada en MB; `vmstat 1 3` muestra tres mediciones, una por segundo, de CPU, memoria, swap y disco.
**Qué es `lastb`:** los intentos de inicio de sesión **fallidos** (lee `/var/log/btmp`). `last` son los exitosos y los reinicios (Día 3).
**Qué es `apachectl configtest` / `sshd -t` / `rsyslogd -N1`:** cada programa tiene su verificador de configuración; se corre **antes** de reiniciar el servicio.
**Qué es `dnf history`:** la lista de instalaciones y actualizaciones con fecha (Día 4). Responde "¿alguien tocó algo antes de que se rompiera?".
**Qué es `setenforce 0`:** apagar SELinux hasta el próximo reinicio. `systemctl stop firewalld`: apagar el firewall. `chmod 777`: darle todo a todos. Los tres "arreglan" el síntoma escondiendo la causa; por eso están prohibidos hoy.

---

## Un ticket es un síntoma, no una causa

**Qué decir:** "un buen técnico no es el que conoce más comandos, es el que **no se salta pasos** cuando está bajo presión."
**Qué señalar:** la fila 6, Verificación: "desde donde el usuario lo ve". `curl localhost` funciona y la web puede seguir caída para toda la oficina, porque `localhost` no pasa por el firewall. Y la fila 7: las cuatro líneas son exactamente lo que van a entregar por cada ticket del reto.

---

## Las capas

**Qué decir:** "si el edificio no tiene luz, no revisás la impresora. Se empieza abajo y se sube solo cuando la capa está sana."
**Qué señalar:** la capa 3 (servicio) va **antes** que la 4 (firewall). El error típico del reto es abrir el firewall cuando el servicio ni siquiera está corriendo: "primero el servicio, después la puerta".

---

## El primer comando de cada capa

No leer la tabla. Tipear los tres comandos del bloque y decir a qué capa pertenece cada uno: `systemctl --failed` (arranque), `sudo ss -tlnp` (red), `df -h` (disco). "Esta tabla es la que van a tener al lado del teclado durante el reto. En el lab que sigue la recorremos entera una vez, en un servidor sano."

---

## Tres reglas para hoy

Leerlas en voz alta, las tres. **La frase para la tercera:** "`setenforce 0` no arregla nada: apaga al que te estaba avisando. En el examen, eso descalifica; en la institución, eso es un incidente de seguridad."
