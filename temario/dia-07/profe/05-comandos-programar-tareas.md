# Comandos — programar tareas (guía del instructor)

Quince minutos, el bloque de concepto más largo del día. Cuatro cosas que tienen que quedar: (1) `cron` repite, `at` es una sola vez, el temporizador de systemd repite integrado con systemd; (2) los cinco campos de una línea de `cron`; (3) la trampa del `PATH` y la salida a un archivo con `>> log 2>&1`; (4) un temporizador son dos archivos, `.service` y `.timer`, y se habilita el `.timer`. Todo lo demás se ve con las manos en los tres labs.

**Qué es `cron`:** el programador de tareas repetitivas de Linux. El servicio se llama `crond` (paquete `cronie`); cada minuto mira qué tareas tocan y las ejecuta.
**Qué es un crontab:** la lista de tareas de un usuario. Cada línea es una tarea: cinco campos de tiempo y el comando. Se guarda en `/var/spool/cron/usuario`, pero **no se edita ahí**: se usa el comando `crontab`.
**Qué son los cinco campos:** minuto (0-59), hora (0-23), día del mes (1-31), mes (1-12), día de la semana (0-7, donde 0 y 7 son domingo). `*` = cualquiera. Después de los cinco, el comando.
**Qué son `,`, `-` y `*/N`:** lista (`8,13` = a las 8 y a las 13), rango (`1-5` = lunes a viernes), paso (`*/15` = cada 15). Funcionan en cualquiera de los cinco campos.
**Qué es `@reboot`:** en vez de los cinco campos: "al arrancar el servidor". También existen `@daily`, `@hourly`, `@weekly`.
**Qué es `crontab -l`, `-e`, `-r`:** listar el mío, editarlo, **borrarlo entero sin preguntar**. La `r` está al lado de la `e` en el teclado: es el accidente clásico. Hoy se usa `crontab ARCHIVO` para no editar en vivo.
**Qué es `crontab ARCHIVO`:** instalar ese archivo como mi crontab. **Reemplaza** todo lo anterior. Ventaja: el archivo queda guardado en el home y se puede reinstalar si alguien hace `-r`.
**Qué es `sudo crontab -u ana -l`:** ver (o con `-e`, editar) el crontab de otro usuario. Solo root.
**Qué son `SHELL=`, `PATH=`, `MAILTO=`:** variables que se ponen arriba del crontab. `SHELL=/bin/bash` para que use Bash (por defecto usa `/bin/sh`). `PATH=…` para que encuentre los scripts. `MAILTO=""` para que no intente mandar correo.
**Qué es la trampa del `PATH`:** `cron` arranca las tareas con un `PATH` mínimo (`/usr/bin:/bin`), sin `~/bin`. Un script que funciona a mano da `command not found` en `cron`. Solución: rutas completas, o `PATH=` arriba del crontab. Es el ticket más común de este tema.
**Qué es `>> /ruta/log 2>&1`:** guardar la salida normal y la de errores de la tarea en un archivo, agregando al final. Sin esto, `cron` manda la salida por correo al usuario, y este servidor no tiene correo: se pierde. Toda tarea de `cron` termina con esto.
**Qué es `/var/log/cron`:** el log de `cron`. Cada ejecución aparece como `(student) CMD (comando)`. Es donde se mira "¿lo ejecutó o no?". Se lee con `sudo`.
**Qué es `/etc/crontab`:** el crontab del sistema. En RHEL viene sin tareas: solo variables y el formato como comentario. Sus líneas (y las de `/etc/cron.d/`) tienen un **sexto campo**, entre el tiempo y el comando: el usuario que ejecuta.
**Qué es `/etc/cron.d/`:** una carpeta donde cada paquete o administrador deja su archivo con tareas de sistema (formato con sexto campo). `crond` los lee solo, sin recargar.
**Qué son `/etc/cron.hourly/`, `cron.daily/`, `cron.weekly/`, `cron.monthly/`:** carpetas con **scripts** (no crontabs). `run-parts` ejecuta todo lo que hay adentro, cada hora, cada día, etc.
**Qué es `run-parts`:** el comando que ejecuta todos los scripts de una carpeta, uno tras otro. `0hourly` lo llama al minuto 01 de cada hora.
**Qué es `anacron`:** el ayudante de `cron` para servidores que no están siempre encendidos: si el servidor estaba apagado cuando tocaba la tarea diaria, la ejecuta al encender. `0anacron` en `cron.hourly` es el que decide si toca.
**Qué es `/etc/cron.deny`:** lista de usuarios que **no** pueden usar `crontab`. Existe vacío por defecto. Si existiera `/etc/cron.allow`, solo los que estén ahí podrían. root siempre puede.
**Qué es `sudo -u jperez comando`:** ejecutar el comando como ese usuario (Día 3). No pide la contraseña de `jperez`.
**Qué es `tee`:** escribe lo que recibe en un archivo y también en pantalla (Día 2). `echo x | sudo tee /etc/archivo` es la forma de escribir un archivo de root: `sudo echo x > /etc/archivo` **no** funciona, porque el `>` lo hace tu shell sin permisos. Con un here-doc (`sudo tee archivo <<'EOF'`) escribe varias líneas.
**Qué es `truncate -s 0 archivo`:** dejar el archivo en cero bytes (vacío) sin borrarlo.
**Qué es PAM y por qué `chage -d` antes de `cron.deny`:** PAM es el sistema que valida las cuentas al entrar (Día 3 lo tocó con las contraseñas). `crontab` también le pregunta a PAM, y una cuenta con cambio de contraseña pendiente (`chage -d 0`, lo que puso el script del Bloque 4) es rechazada con un mensaje de PAM en vez del de `cron.deny`. `sudo chage -d "$(date +%F)" jperez` le pone fecha de hoy y lo deja normal.
**Qué es `at`:** programar **una** tarea para **una** vez: "dentro de 2 minutos", "a las 17:30", "mañana a las 02:00". El servicio es `atd`. En RHEL 9 el paquete `at` puede no estar instalado.
**Qué es `at now + 2 minutes`:** abre el prompt `at>`. Ahí se escriben los comandos, uno por línea, y se cierra con `Ctrl+D` (fin de entrada). Responde `job N at fecha`.
**Qué es `echo "comando" | at 17:30`:** lo mismo en una línea: el comando entra por la tubería.
**Qué son `atq` y `atrm`:** `atq` lista los trabajos pendientes (número, fecha, usuario). `atrm N` borra el trabajo `N`.
**Qué es `rpm -q at`:** pregunta si el paquete está instalado (Día 4). Con `|| sudo dnf install -y at`: "si no, instalalo".
**Qué es un temporizador de systemd (timer):** la forma de systemd de programar tareas. Son **dos** unidades con el mismo nombre: `nombre.service` (qué se ejecuta) y `nombre.timer` (cuándo). Se habilita y arranca **el timer**; el timer dispara el service.
**Qué es `Type=oneshot`:** un service que ejecuta el comando, termina, y listo. Distinto del `Type=simple` de `monitor.service` del Día 4, que se quedaba corriendo.
**Qué es `ExecStart=`:** el comando que ejecuta el service, con ruta completa (Día 4).
**Qué es `OnCalendar=`:** cuándo se dispara el timer, en el formato de systemd: `DíaSemana Año-Mes-Día Hora:Min:Seg`, con `*` como comodín. `*:0/10` = cada 10 minutos. `Mon..Fri 02:00` = lunes a viernes a las 02:00.
**Qué es `Persistent=true`:** si el servidor estaba apagado cuando tocaba, la ejecuta al arrancar. Es lo que `cron` de usuario no hace (y `anacron` sí, para las del sistema).
**Qué es `[Install] WantedBy=timers.target`:** lo que hace que `systemctl enable` funcione: el timer se engancha al grupo de temporizadores del sistema (Día 4, `WantedBy=multi-user.target` para servicios).
**Qué es `daemon-reload`:** decirle a systemd que relea los archivos de `/etc/systemd/system/` (Día 4). Obligatorio después de crear o editar una unidad.
**Qué es `systemctl enable --now`:** habilitar para el arranque y arrancar ahora (Día 4).
**Qué es `systemctl list-timers`:** todos los timers con su próxima ejecución (`NEXT`), cuánto falta (`LEFT`), la última (`LAST`). Con el nombre de un timer, solo ese.
**Qué es `systemd-analyze calendar "expresión"`:** comprueba una expresión de `OnCalendar` y dice cuándo sería la próxima ejecución. `--iterations=3` muestra las próximas tres. Se usa **antes** de escribir el timer.
**Qué es `journalctl -u nombre.service`:** el log de esa unidad (Día 4). Lo que el script imprime va ahí solo, sin `>>`.
**Qué es `systemctl status nombre.timer`:** estado del timer: `active (waiting)` = esperando la próxima; `Trigger:` = cuándo.
**Qué es `systemd-tmpfiles`:** el mecanismo de systemd que crea y limpia carpetas temporales según reglas. Las reglas del sistema están en `/usr/lib/tmpfiles.d/`; las propias van en `/etc/tmpfiles.d/`. `systemd-tmpfiles-clean.timer` las aplica una vez al día: así se limpia `/tmp` (10 días) y `/var/tmp` (30 días).
**Qué es una línea de `tmpfiles.d`:** `tipo ruta permisos dueño grupo edad`. Tipo `q` (o `d`): crear la carpeta si no existe y borrar lo más viejo que la edad. Las de tipo `x`/`X` son excepciones: carpetas que no se tocan.
**Qué es `grep -v "#"`:** mostrar las líneas que **no** contienen `#` (`-v` invierte, Día 2). Saca los comentarios del archivo.

---

## Tres herramientas, tres preguntas

**Qué decir:** "¿se repite? `cron`. ¿Una sola vez? `at`. ¿Se repite y la quiero en el journal, o que se recupere si el servidor estaba apagado? Temporizador de systemd. Las tres son válidas; lo que no se hace es programar la misma tarea en dos".

---

## `cron`: anatomía de una línea

**Qué decir, leyendo el diagrama de izquierda a derecha:** "minuto, hora, día del mes, mes, día de la semana, comando".
**Qué señalar:** en la tabla, `*/15` (cada 15) y `1-5` (lunes a viernes). Y en el diagrama, el final de la línea: ruta completa y `>> log 2>&1`. **La frase:** *"si escriben `hola.sh` a secas en `cron`, no lo encuentra. `cron` no conoce su `~/bin`. Ruta completa, siempre."*

---

## El crontab del usuario

**Qué señalar:** `crontab -r` borra sin preguntar. Hoy se instala desde un archivo (`crontab ~/mi-crontab`) para que el archivo quede guardado. Las tres variables de arriba: leerlas y decir qué evita cada una (`SHELL` = que use Bash; `PATH` = que encuentre los scripts; `MAILTO=""` = que no intente mandar correo).

---

## `cron` del sistema

Los cuatro comandos, rápido.
**Qué señalar:** en `cat /etc/crontab`, la última línea comentada dice `user-name` entre el tiempo y el comando: es el sexto campo. En `0hourly`, ese campo dice `root`. `ls /etc/cron.hourly` muestra `0anacron`: "de ahí sale que las tareas diarias del sistema se ejecuten aunque el servidor haya estado apagado". `sudo tail -3 /var/log/cron`: ahora casi vacío; en el lab lo van a ver llenarse.

---

## `at`: una sola vez

**Qué decir:** "`at` es para 'reiniciá el servicio a las 2 de la mañana, pero solo hoy'. Se escribe el comando, se cierra con `Ctrl+D`, y `atq` muestra que quedó". No demostrar acá.

---

## Temporizador de systemd

**Qué decir, señalando el diagrama:** "dos archivos con el mismo nombre. El `.service` dice **qué**: `Type=oneshot`, corre y termina. El `.timer` dice **cuándo**: `OnCalendar`. Se habilita el **timer**". Comparar con el Día 4: `monitor.service` era `Type=simple` y corría siempre; este corre cada 10 minutos y termina.
**Qué señalar:** `Persistent=true`, y la tabla de `OnCalendar`: `Mon..Fri 02:00` es la que va en el reto. `systemd-analyze calendar`: "antes de escribir el timer, prueben la expresión con esto". El `systemctl list-timers --no-pager | head -5`: los timers que RHEL ya trae (`dnf-makecache`, `logrotate`, `systemd-tmpfiles-clean`).

---

## Quién limpia `/tmp`

`grep -v "#" /usr/lib/tmpfiles.d/tmp.conf`.
**Qué decir:** "esto es lo que borra `/tmp` cada 10 días en RHEL 9. Una regla por línea: tipo, ruta, permisos, dueño, grupo, edad. Las propias van en `/etc/tmpfiles.d/`". Una frase y seguir: no hay lab de esto.
