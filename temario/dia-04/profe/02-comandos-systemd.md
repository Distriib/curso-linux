# Comandos — servicios con systemd (guía del instructor)

Diez minutos. **El** concepto del día es uno solo: "¿está corriendo ahora?" y "¿arranca con el sistema?" son dos preguntas **independientes**, con comandos distintos (`start`/`stop` vs `enable`/`disable`). Si eso queda, el resto se practica. Lo segundo: después de crear o editar un unit file, `daemon-reload` siempre.

**Qué es systemd:** el proceso 1 (lo vieron en el Bloque 1) y además el administrador de todo lo que arranca después: servicios, montajes, temporizadores. Reemplazó a los scripts de arranque viejos (`/etc/init.d`, RHEL 6).
**Qué es una unidad (unit):** cada cosa que systemd administra. Es un archivo de texto; la extensión dice el tipo: `sshd.service` es un servicio, `multi-user.target` es un target.
**Qué es un `.service`:** un programa que systemd arranca, vigila y detiene: `sshd`, `httpd`, `chronyd`.
**Qué es un `.target`:** un grupo de unidades; el "modo" en que arranca el sistema. `multi-user.target` = servidor con red y consola de texto (lo nuestro). `graphical.target` = lo mismo más escritorio. Cuando el target arranca, arranca todo lo que él "quiere" (`wants`).
**Qué es un `.timer`:** ejecuta un `.service` según calendario: `logrotate.timer` corre `logrotate.service` una vez al día. Es la alternativa moderna a cron (Día 7).
**Qué es un `.socket`:** escucha en un puerto y arranca el servicio solo cuando alguien se conecta. `cockpit.socket` (puerto 9090, la consola web) es el ejemplo en la VM.
**Qué es un `.mount`:** un punto de montaje (dónde se "cuelga" un disco en el árbol de carpetas). Se generan solos desde `/etc/fstab` (Día 6). Solo se nombra.
**Qué es un paquete:** software empaquetado para instalar con `dnf`. Detalle en el Bloque 4; acá alcanza con "lo que instala `dnf install`".
**Qué es `/usr/lib/systemd/system/`:** donde los paquetes ponen sus unit files. No se editan: una actualización los pisa. **Qué es `/etc/systemd/system/`:** donde el administrador pone los suyos. Si hay dos con el mismo nombre, gana el de `/etc`.
**Qué es `start`/`stop`/`restart`:** arrancar, detener, y detener-y-arrancar **ahora**. **Qué es `reload`:** pedirle al servicio que relea su configuración sin cortar lo que está atendiendo. Por debajo suele ser un `kill -HUP` a su proceso principal. No todos lo soportan.
**Qué es `enable`/`disable`:** que arranque (o no) **con el sistema**. No arranca ni detiene nada ahora. `enable --now` = `enable` + `start`.
**Qué es `is-active`/`is-enabled`:** responden con una palabra: `active`/`inactive`, `enabled`/`disabled`/`masked`. Sirven para mirar rápido y para scripts.
**Qué es un enlace simbólico:** un archivo que solo apunta a otro; en `ls -l` se ve con `->`. `enable` no hace más que crear uno en `/etc/systemd/system/multi-user.target.wants/` apuntando al unit file; `disable` lo borra. Eso es todo el misterio.
**Qué es `mask`:** prohibir la unidad: crea un enlace `/etc/systemd/system/nombre.service -> /dev/null`. Como `/etc` gana sobre `/usr/lib`, la unidad "es" `/dev/null`: nadie puede arrancarla, ni a mano ni como dependencia de otra. `unmask` borra el enlace. `mask` **no** detiene un servicio que ya corre: por eso antes se hace `stop`.
**Qué es `Loaded:`:** primera línea de `status`: de qué archivo salió la unidad, si está `enabled`/`disabled`/`masked`, y el `preset`.
**Qué es `preset`:** lo que RHEL haría por defecto al instalar el paquete: `enabled` para sus servicios base (`sshd`), `disabled` para lo demás (`httpd`) y para lo que uno escribe.
**Qué es `Active:`:** si corre ahora. `active (running)` corriendo; `inactive (dead)` detenido; `failed` murió mal; `activating` arrancando.
**Qué es `Main PID`:** el proceso principal del servicio, el que lanzó `ExecStart`. **Qué es `CGroup`:** la lista de **todos** los procesos que systemd considera parte del servicio, hijos incluidos. `stop` los mata a todos; por eso systemd no deja procesos huérfanos como pasaba con los scripts viejos.
**Qué es `●` / `○`:** el puntito delante del nombre en `status`: relleno = activo, vacío = inactivo. Rojo = falló.
**Qué es `list-units`:** qué unidades están cargadas **ahora** (columna `ACTIVE`). `--type=service --state=running` = solo servicios corriendo. **Qué es `list-unit-files`:** qué archivos de unidad hay y su estado de **arranque** (`STATE`: `enabled`, `disabled`, `static` = no tiene `[Install]`, lo arranca otra unidad; `masked`). **Qué es `--failed`:** solo las que fallaron; lo deseable es `0 loaded units listed`.
**Qué es `systemctl cat`:** muestra el archivo de la unidad tal cual, con su ruta en la primera línea. **Qué es `systemctl show -p MainPID --value`:** una sola propiedad, sin adornos. Sirve para comparar el PID antes y después de `reload`/`restart`.
**Qué es `list-timers`:** los temporizadores, cuándo corrieron y cuándo vuelven a correr.
**Qué es `get-default`/`set-default`:** con qué target arranca el sistema. `set-default` cambia un enlace: `/etc/systemd/system/default.target`. En la VM sin escritorio, `graphical.target` arrancaría igual en texto, pero en el examen RHCSA lo piden.
**Qué es `daemon-reload`:** systemd relee todos los unit files del disco. Si creás o editás uno y no lo hacés, systemd sigue con la versión vieja (o dice `Unit not found`). Es el olvido número uno.
**Qué es el pager:** cuando la salida no entra en la pantalla, `systemctl` la abre en un visor (`less`): flechas para moverse, `q` para salir. `--no-pager` lo evita; hace falta para pegar en el chat.
**Qué es un unit file:** el archivo de texto de una unidad. Tres secciones entre corchetes: `[Unit]` qué es y con qué se relaciona; `[Service]` cómo se ejecuta; `[Install]` qué hace `enable`.
**Qué es `Description=`:** el nombre legible que sale en `status`. **Qué es `After=network.target`:** arrancar después de que la red esté lista. Es orden, no exigencia.
**Qué es `Type=simple`:** el proceso que lanza `ExecStart` **es** el servicio; cuando ese proceso muere, el servicio murió. Es el default y el que sirve para un script que corre en bucle. (`sshd` usa `Type=notify`: avisa a systemd cuando está listo. Solo nombrarlo.)
**Qué es `ExecStart=`:** el comando que arranca el servicio. Ruta **completa** obligatoria. **Qué es `ExecReload=`:** qué hacer cuando piden `reload`.
**Qué es `Restart=on-failure`:** si el proceso termina mal (código distinto de 0, o lo matan con una señal), systemd lo relanza. Un `systemctl stop` limpio **no** cuenta como fallo. `RestartSec=5` = esperar 5 segundos antes de relanzar.
**Qué es `KillMode=process`:** al detener, matar solo el proceso principal y no sus hijos. `sshd` lo usa: por eso `restart sshd` no corta las sesiones abiertas.
**Qué es `WantedBy=multi-user.target`:** "cuando hagan `enable`, colgame de `multi-user.target`". Es lo que le dice a `enable` en qué carpeta `.wants/` crear el enlace.
**Qué es `systemd-analyze verify`:** revisa la sintaxis de un unit file. No imprime nada si está bien. Es el `visudo -c` de systemd.
**Qué es `httpd`:** el servidor web Apache. Se instala hoy porque es el servicio ideal para practicar: no viene arrancado ni habilitado, y se puede probar con `curl`. **Qué es `/var/www/html/index.html`:** la página principal que sirve. **Qué es `curl http://localhost`:** pide la página a la propia VM y la muestra en la terminal.
**Qué es `tee`:** escribe en un archivo lo que recibe por la tubería y además lo muestra en pantalla. `echo "..." | sudo tee /var/www/html/index.html` es la forma de escribir un archivo donde solo root puede, porque `sudo echo > archivo` no funciona (la redirección la hace tu shell, sin sudo).
**Qué es `dnf install -y httpd`:** instalar el paquete `httpd` respondiendo sí a todo. Se explica a fondo en el Bloque 4.
**Qué es `timeout 3 comando`:** corre el comando y lo corta a los 3 segundos. Sirve para probar un programa que no termina solo.
**Qué es `#!/bin/bash`:** la primera línea de un script: dice con qué programa ejecutarlo. Ya lo vieron ayer en `hola.sh`.
**Qué es `while true` / `do` / `done`:** "repetir para siempre lo que está entre `do` y `done`". Es la única forma de que un programa no termine, que es lo que necesita un servicio. Se ve a fondo el Día 7.
**Qué es `logger -t monitor -p local0.info`:** escribe un mensaje en el log del sistema. `-t` es la etiqueta (quién habla); `-p` es `facilidad.prioridad`: `local0` es una "casilla" reservada para uso propio, `info` es la gravedad. Sin mensaje, `logger` toma lo que le llega por la tubería. Todo esto se explica a fondo en el Bloque 3; acá alcanza con "el programa escribe en el log".
**Qué es `journalctl -u monitor`:** el log de esa unidad. `-n 3` últimas 3 líneas; `-f` sigue en vivo como `tail -f`; `--no-pager` sin visor. Ayer usaron `journalctl -t sudo`. Bloque 3 a fondo.
**Qué es `chronyd`:** el servicio que mantiene la hora sincronizada con servidores de internet (ayer lo reiniciaron con `sudo`). **Qué es NTP:** el protocolo para eso (*Network Time Protocol*). **Qué es `chronyc sources`:** con qué servidores de hora está hablando `chronyd`. **Qué es `timedatectl`:** hora local, hora universal (UTC), zona horaria y si está sincronizado.
**Qué es la zona horaria:** el desplazamiento respecto de UTC. Panamá es `America/Panama`, UTC-5, sin horario de verano; se ve como `EST, -0500`.

---

## systemd es PID 1 y maneja todo lo que arranca después

La tabla de tipos: señalar `.service` y `.target`; los otros solo nombrarlos ("los van a ver en `list-timers`"). Las dos carpetas: **la frase exacta:** *"lo de `/usr/lib` es del paquete; lo de `/etc` es tuyo. Si hay dos iguales, gana `/etc`. Eso es lo que hace funcionar a `mask`."*

---

## Las dos preguntas

`systemctl status sshd`, `is-active sshd`, `is-enabled sshd`.
**Qué decir, despacio, es lo más importante del bloque:** *"¿corre ahora? es `start` y `stop`. ¿Arranca con el sistema? es `enable` y `disable`. Son independientes. Un servicio puede estar corriendo y deshabilitado: anda perfecto hasta que alguien reinicia. Ese es el ticket 'ayer funcionaba'."*
**Qué señalar:** en `status`, las dos líneas: `Loaded: ... enabled` responde la segunda pregunta; `Active: active (running)` responde la primera.

---

## Leer `status` completo

**Qué señalar** en el ejemplo de `httpd`: `Loaded` (de dónde, `enabled`, `preset: disabled`), `Active`, `Main PID`, `CGroup` con la lista de procesos, y abajo el log. **Qué decir:** "la mitad de los diagnósticos terminan en esas últimas líneas: es el log del servicio, sin buscarlo."

---

## `systemctl` — lo que se usa

No leer la tabla. Dos: `--failed` ("lo primero que corro en un servidor que no conozco") y `daemon-reload` ("después de tocar un unit file, siempre; systemd avisa si te olvidás").

---

## Anatomía de un unit file

Leer el diagrama de arriba a abajo, una frase por línea. Lo que tiene que quedar: `ExecStart` con ruta completa, `Restart=on-failure` ("si muere mal, vuelve solo: eso `nohup` no lo da"), `WantedBy=multi-user.target` ("le dice a `enable` dónde colgarlo"). Cierre: `verify` y `daemon-reload`.

---

## La hora

`timedatectl` y `chronyc sources`.
**Qué decir:** "los logs se ordenan por hora. Con la hora mal, no podés cruzar el log del servidor con el del firewall ni con nada. Es lo primero que se revisa."
**Qué señalar:** `System clock synchronized: yes`, `NTP service: active`, `Time zone: America/Panama`. Si alguien tiene otra zona: `sudo timedatectl set-timezone America/Panama`.
