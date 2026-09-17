# Comandos — primer script (guía del instructor)

Diez minutos, tipeando mientras explicás. Tres cosas que tienen que quedar: (1) un script es un archivo con comandos y una primera línea `#!/bin/bash`; (2) para ejecutarlo por nombre hace falta permiso `x` y que esté en `~/bin`; (3) las variables se escriben `NOMBRE=valor` sin espacios y se leen con `$NOMBRE`, entre comillas dobles. Lo demás se practica en el lab.

**Qué es un script:** un archivo de texto con comandos, uno por línea. Bash lo lee de arriba a abajo y ejecuta cada línea como si la hubieras tipeado.
**Qué es el shebang:** la primera línea, `#!/bin/bash`. Le dice al sistema con qué programa ejecutar el archivo. Sin ella, el sistema adivina. Se pronuncia "shebang" porque `#` es *sharp* y `!` es *bang*.
**Qué es un intérprete:** el programa que lee el script y lo ejecuta. Acá es `bash`, el mismo que atiende la terminal.
**Qué es un comentario:** una línea que empieza con `#`. Bash la ignora. La cabecera de todo script dice qué hace, cómo se usa y qué códigos devuelve, para el que lo lea dentro de un año.
**Qué es el permiso de ejecución:** la `x` de `rwx` (Día 3). Sin ella, `./hola.sh` da `Permission denied`. `chmod +x hola.sh` la agrega para todos. `bash hola.sh` no la necesita porque el que se ejecuta es `bash`, y el script es solo un archivo que lee.
**Qué es `./`:** "en esta carpeta". `./hola.sh` = el `hola.sh` que está acá. Sin el `./`, Bash busca en el `PATH`, no en la carpeta actual.
**Qué es el `PATH`:** la lista de carpetas, separadas por `:`, donde Bash busca un comando cuando lo escribís sin ruta. `ls` funciona porque `/usr/bin` está en el `PATH`. `echo "$PATH"` la muestra.
**Qué es `~/bin`:** la carpeta `/home/student/bin`. RHEL 9 la pone en el `PATH` de cada usuario (lo hace `~/.bashrc`, un archivo que se lee al abrir sesión) aunque la carpeta no exista. Es el lugar para los scripts propios.
**Qué es `/usr/local/bin`:** la carpeta para scripts que va a ejecutar root, `cron` del sistema o systemd. También está en el `PATH`. Se usa en el Bloque 4.
**Qué es una variable:** un nombre que guarda un valor. `NOMBRE="Ana"` guarda; `$NOMBRE` lee. Por convención, MAYÚSCULAS.
**Qué son las comillas dobles vs simples:** dentro de `"..."`, Bash reemplaza `$NOMBRE` por su valor. Dentro de `'...'`, no toca nada: `'$NOMBRE'` imprime literalmente `$NOMBRE`.
**Qué es `${NOMBRE}`:** lo mismo que `$NOMBRE`, con llaves. Hace falta cuando sigue texto pegado: `${NOMBRE}_reporte.txt`. Sin llaves, Bash buscaría una variable llamada `NOMBRE_reporte`.
**Qué es `$(comando)`:** "la salida de este comando". `FECHA=$(date +%F)` ejecuta `date` y guarda lo que imprimió. Se llama sustitución de comandos. Se puede poner adentro de un `echo "..."`.
**Qué es `$(( ))`:** una cuenta con números enteros. `$((7 * 6))` es `42`. `$((10 / 3))` es `3`: no hay decimales.
**Qué es `read -p`:** pide un dato por teclado. `read -p "Nombre: " TECNICO` muestra `Nombre: `, espera que escriban algo y lo guarda en `TECNICO`. En un script que corre solo (con `cron`) no hay teclado: ahí todo tiene que venir por argumentos.
**Qué es un argumento:** lo que escribís después del nombre del script, separado por espacios. `args.sh uno dos` tiene dos argumentos. Si un argumento tiene espacios, va entre comillas: `"Sala de servidores"` es uno solo.
**Qué son `$0`, `$1`, `$#`, `$@`:** variables que Bash llena solo. `$0` = nombre del script, `$1` = primer argumento, `$2` = segundo, `$#` = cuántos llegaron, `$@` = todos juntos.
**Qué es `$?`:** el código de salida del último comando. `0` = salió bien. Cualquier otro número = falló. Se lee justo después del comando; el siguiente comando lo pisa.
**Qué es un here-doc:** la forma de escribir varias líneas en un archivo sin abrir el editor. `cat > archivo <<'EOF'` dice "todo lo que sigue, hasta la línea que diga `EOF`, va al archivo". `EOF` es una palabra cualquiera (*end of file*); podría ser `FIN`. Las comillas en `<<'EOF'` hacen que Bash **no** reemplace los `$` del contenido: el `$NOMBRE` llega al script tal cual, que es lo que queremos. El Día 2 lo usaron para `genera-log.sh` sin explicarlo; hoy sí.
**Qué es `uname -r`:** la versión del kernel. **`uptime -p`:** hace cuánto está encendido, en palabras. **`df -h / | tail -1`:** la línea del disco raíz, sin encabezado. Los tres salen en `reporte.sh`.

---

## Qué es un script

**Qué decir:** "todo lo que escribieron en la terminal estos seis días funciona adentro de un archivo. Un script es eso: los comandos guardados para no volver a tipearlos".
**Qué señalar:** en el diagrama, la línea 1 es el shebang; la línea 2 es un comentario; las otras dos son Bash común.

---

## Escribirlo sin abrir un editor

El `cat > /tmp/prueba.sh <<'EOF'` y el `bash /tmp/prueba.sh`.
**Qué decir:** "hoy vamos a escribir scripts con `cat >` y un here-doc. Yo pego el bloque en el chat, ustedes lo copian entero, con la línea `EOF` final incluida". Pegar cada bloque en el chat antes de que lo tipeen: son diez líneas por script y no hay tiempo para que las tipeen a mano.
**Qué señalar:** la última línea, `EOF`, sola, sin espacios adelante. Si falta, la terminal se queda esperando (el prompt no vuelve): `Ctrl+C` y de nuevo.

---

## Tres formas de ejecutarlo

`echo "$PATH"`.
**Qué decir:** "`bash hola.sh` sirve para probar; `hola.sh` a secas es la forma final, y para eso hace falta permiso `x` y que esté en `~/bin`".
**Qué señalar:** `/home/student/bin` en la salida del `PATH`, aunque todavía no exista la carpeta. Cuando la creen en el lab, todo lo que pongan ahí con `x` se ejecuta por nombre desde cualquier lugar.

---

## Variables

**Qué decir despacio:** *"`NOMBRE=valor`, sin espacios. Si escriben `NOMBRE = valor`, Bash cree que `NOMBRE` es un comando y da `command not found`. Es el error número uno del día."*
**Qué señalar:** las dos filas de comillas. Dobles reemplazan, simples no. Regla que se repite todo el día: variables con rutas o texto, siempre entre comillas dobles: `"$RUTA"`. Si la ruta tiene un espacio y no hay comillas, el comando recibe dos argumentos y se rompe.

---

## Lo que el script recibe

**Qué señalar:** en el diagrama, `"Sala de servidores"` es un solo argumento gracias a las comillas; por eso `$#` da 3 y no 5. `$0` es el nombre del script tal como se lo llamó (en el lab va a salir con la ruta completa, porque lo encuentra por el `PATH`).

Los dos `ls; echo "código: $?"`.
**Qué decir:** "`0` es 'bien'. `ls` de algo que no existe da `2`. En el Bloque 2 nuestros scripts van a devolver sus propios códigos, y `cron` los va a mirar".
