# Comandos — scripts de administración (guía del instructor)

Cinco minutos. Una sola idea: los dos scripts de este bloque no tienen nada nuevo de Bash (`if`, `while read`, `$((…))`, `exit`), pero sí comandos de administración que hay que nombrar antes de leerlos. No demostrar acá; se hace todo en los labs. Los scripts se pegan en el chat: la meta es que los lean y entiendan qué hace cada parte, no que los tipeen.

**Qué es `$EUID`:** el número de usuario con el que está corriendo el script (*effective user ID*). Con `sudo` es `0` (root). `[[ $EUID -ne 0 ]]` = "no soy root". Es la forma de que un script que crea usuarios se niegue a correr sin `sudo`.
**Qué es `getent group nombre`:** consulta si el grupo existe (Día 3). Código `0` si existe, `2` si no. Con `> /dev/null` se tira la salida y se usa solo el código: `if ! getent group "$grupo" > /dev/null` = "si el grupo no existe".
**Qué es `groupadd`:** crea un grupo (Día 3). Sin `-g`, el sistema elige el número.
**Qué es `id usuario`:** muestra UID y grupos (Día 3). Si el usuario no existe, falla. `> /dev/null 2>&1` tira salida y error: solo importa el código.
**Qué es `useradd -m -G grupo -c "desc" usuario`:** crear el usuario (Día 3). `-m` crea su carpeta, `-G` lo mete en ese grupo como suplementario (conserva su grupo privado), `-c` la descripción. Sin `-u`: el sistema asigna el próximo número libre (2005 en adelante, después de `laura`).
**Qué es `chpasswd`:** pone contraseñas leyendo `usuario:clave` por la entrada. `echo "ana:Clave" | chpasswd`. Es la versión para scripts del `passwd --stdin` del Día 3.
**Qué es `chage -d 0`:** marca la contraseña como "cambiada el día 0": el sistema obliga a cambiarla en el próximo ingreso (Día 3, Lab 2). Así la clave inicial `Cambiar.2026` dura una sola sesión.
**Qué es `continue`:** dentro de un bucle, "saltá el resto de esta vuelta y seguí con la siguiente". En el script: si el usuario ya existe, no se hace nada más con esa línea.
**Qué es idempotente:** que ejecutarlo dos veces deja el mismo resultado que una. El script lo es porque pregunta "¿existe?" antes de crear. Un `useradd` suelto no lo es: la segunda vez falla.
**Qué es `secure_path`:** la lista de carpetas donde `sudo` busca comandos. Es otra que tu `PATH` y no incluye `~/bin` ni `/usr/local/bin`. Por eso `sudo crear-usuarios.sh` da `command not found` y hay que escribir `sudo ~/bin/crear-usuarios.sh`.
**Qué es `df --output=target,pcent`:** `df` mostrando solo dos columnas: punto de montaje (`target`) y porcentaje usado (`pcent`). Sin las demás, es fácil de leer con `read`.
**Qué son `tmpfs`, `devtmpfs`, `efivarfs`:** sistemas de archivos que viven en memoria, no en disco (`/dev`, `/run`, `/dev/shm`, y las variables de arranque UEFI). `-x tipo` los excluye: no tiene sentido alertar por ellos.
**Qué es `tail -n +2`:** "desde la línea 2 hasta el final". Sirve para saltar el encabezado. Distinto de `tail -2` (las últimas dos).
**Qué es `tr -d '%'`:** borrar (`-d`) el carácter `%` de lo que entra. `18%` → `18`. Hace falta porque `18%` no es un número para `[[ -gt ]]`.
**Qué es `mktemp`:** crea un archivo temporal vacío en `/tmp` con nombre único (`/tmp/tmp.Xa8fk2`) y lo imprime. `TMP=$(mktemp)` guarda ese nombre. Al final, `rm -f "$TMP"`. Se usa en vez de un nombre fijo para que dos ejecuciones no se pisen.
**Qué es `logger -p local0.warning`:** `logger` con prioridad (`-p`) de aviso, en la facilidad `local0` (Día 4, la que usó `monitor.sh`). Las alertas quedan marcadas como `warning` en el journal; los `OK` no se mandan a `logger`, solo al archivo propio.
**Qué es `/var/log/monitor-disco.log`:** el archivo de log propio del script. Lo crea la primera vez que escribe (como root). Una línea por sistema de archivos revisado, con fecha.
**Qué es un umbral:** el porcentaje a partir del cual se avisa. Por defecto 80; se cambia con el primer argumento.
**Qué es la etiqueta de seguridad de un archivo:** SELinux (Día 8) le pone a cada archivo una etiqueta según dónde está. Un archivo que nace o se **copia** a `/usr/local/bin` recibe la etiqueta de programa; uno que se **mueve** con `mv` desde `/home` se lleva la etiqueta de "archivo de usuario" y systemd puede negarse a ejecutarlo. Regla de hoy: `cp`, no `mv`. Se explica bien el Día 8.
**Qué es `chmod 755`:** `rwxr-xr-x` (Día 3): root escribe, todos leen y ejecutan. Lo que corresponde a un programa del sistema.

---

## Altas de usuarios

**Qué decir:** "es el `useradd` del Día 3, cuatro veces, sin tipearlo cuatro veces. El script lee una línea del archivo, pregunta si el grupo existe, pregunta si el usuario existe, y crea lo que falte".
**Qué señalar:** la fila de `useradd`: es la misma del Día 3 con `-G` en vez de `usermod -aG`. Y la última línea: `sudo` con ruta completa.

---

## Alerta de disco

El `df --output=…`: tipearlo para que vean las dos columnas.
**Qué decir:** "el script lee esa salida línea por línea con `while read punto uso`, le saca el `%` al porcentaje y lo compara con el umbral".
**Qué señalar:** `tail -n +2` saca el encabezado. `mktemp` es para guardar esa salida en un archivo temporal y leerla con `done < "$TMP"`, la misma forma del Bloque 3.

---

## Dónde va cada script

**La frase que hay que decir:** *"`~/bin` es para lo que ejecutás vos. `/usr/local/bin` es para lo que ejecuta el servidor: root, `cron` del sistema, systemd. Y se **copia** con `cp`, no se mueve con `mv`."* El porqué completo es del Día 8; hoy alcanza con la regla.
