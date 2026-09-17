# Comandos — un script que decide (guía del instructor)

Diez minutos. Tres cosas que tienen que quedar: (1) un script termina con un código, `0` es bien, y se elige con `exit N`; (2) `if [[ prueba ]]; then ... fi`, con las pruebas `-d`, `-f`, `-z`, `-ne`; (3) `bash -x` muestra qué hizo el script paso a paso. El resto se ve en el lab.

**Qué es el código de salida:** el número que deja todo comando al terminar. `0` = bien; `1`, `2`, … = falló. Lo vieron en el Bloque 1 con `$?`. Un script también deja uno: el del último comando que ejecutó, o el que vos pongas con `exit N`.
**Qué es `exit N`:** termina el script en ese momento con el código `N`. Convención: `1` para "mal uso" (falta un argumento), `2` en adelante para errores propios. `cron` y systemd anotan si la tarea terminó con algo distinto de `0`.
**Qué es `if … then … fi`:** "si la prueba da verdadero, hacé esto". `then` abre el bloque, `fi` lo cierra (`if` al revés). `elif` es "si no, probá esta otra"; `else` es "si ninguna". El `;` antes de `then` es porque `then` va en la misma línea.
**Qué es `[[ ]]`:** la prueba. Adentro va la condición. Los espacios después de `[[` y antes de `]]` son obligatorios: `[[-d $X]]` es error de sintaxis. Es la versión moderna de `test` / `[ ]`; en Bash se usa `[[ ]]`.
**Qué son `-e`, `-f`, `-d`, `-x`:** preguntas sobre un archivo: existe, es un archivo común, es una carpeta, es ejecutable. Se escriben delante de la ruta: `[[ -d "$RUTA" ]]`.
**Qué es `-z`:** "está vacía" (*zero length*). `[[ -z "$X" ]]` es verdadero si `X` no tiene nada. Sirve para detectar que faltó un argumento.
**Qué son `-eq`, `-ne`, `-gt`, `-lt`, `-ge`, `-le`:** comparaciones de números: igual, distinto, mayor, menor, mayor o igual, menor o igual (*equal, not equal, greater than, less than, greater or equal, less or equal*). Para texto se usa `==` y `!=`; para números, estas. `[[ 10 > 9 ]]` compara como texto y da falso: siempre `-gt` con números.
**Qué es `$# -ne 1`:** "la cantidad de argumentos no es 1". Es la validación típica de la primera línea de un script.
**Qué es `!`:** "lo contrario". `[[ ! -d "$X" ]]` = "no es una carpeta". `if ! comando` = "si el comando falla".
**Qué es `&&`:** "y si salió bien". `a && b` ejecuta `b` solo si `a` terminó con `0`. `[[ -x "$RUTA" ]] && echo "ejecutable"` es un `if` de una línea.
**Qué es `||`:** "o si falló". `a || b` ejecuta `b` solo si `a` terminó distinto de `0`. `rpm -q at || sudo dnf install -y at` = "si no está instalado, instalalo".
**Qué es `>&2`:** mandar el mensaje a la salida de errores (stderr, la número 2) en vez de a la salida normal (stdout, la 1). Día 2: `2>` redirige la de errores. Los mensajes de error de un script van con `>&2` para que no se mezclen con los datos si alguien hace `script | otro`.
**Qué es `2> /dev/null`:** tirar los errores. `/dev/null` es el agujero negro del Día 2: lo que entra desaparece.
**Qué es `set -e`:** una orden al principio del script: "si un comando falla, terminá acá". Sin ella, Bash sigue con la línea siguiente como si nada y el script termina con `0` aunque haya fallado en el medio. Con ella, el código final es el del comando que falló.
**Qué es `bash -n`:** revisa la sintaxis del script sin ejecutarlo. No imprime nada si está bien. Detecta el `fi` que falta, las comillas sin cerrar.
**Qué es `bash -x`:** ejecuta el script mostrando cada comando antes de correrlo, con `+` delante y las variables ya reemplazadas. `++` es un comando que estaba dentro de `$( )`. Es la herramienta para "el script hace algo raro y no sé por qué".
**Qué es `logger`:** escribe una línea en el log del sistema (Día 4). `logger -t revisar "mensaje"`: `-t` es la etiqueta (*tag*) con la que se busca después.
**Qué es `journalctl -t`:** leer del journal (Día 4) solo las líneas con esa etiqueta. `-n 1` = la última. `--no-pager` = imprimir directo, sin abrir el visor.
**Qué es `stat -c %s`:** el tamaño de un archivo en bytes, solo el número. `stat` lo vieron el Día 2; `-c %s` le pide un solo dato.
**Qué es `ls -A | wc -l`:** contar las entradas de una carpeta (`-A` incluye las ocultas, sin `.` ni `..`).
**Qué es `$USER`:** variable que el sistema llena con tu nombre de usuario. Sale en el mensaje de `logger`.

---

## Códigos de salida

**Qué decir:** "un script que falla y termina con `0` es un script mentiroso: a las 3 de la mañana `cron` lo va a dar por bueno".
**Qué señalar:** la tabla de convención. En el lab, `revisar.sh` devuelve `1` si falta el argumento y `2` si la ruta no existe: dos fallas distintas, dos números distintos.

---

## Anatomía de un `if`

**Qué decir, leyendo el diagrama de arriba a abajo:** "si es carpeta, esto; si no, si es archivo, esto otro; si no, esto; `fi` cierra".
**Qué señalar:** el `;` antes de cada `then`, y los espacios dentro de `[[ ]]`. Los dos errores de sintaxis del día están ahí.

---

## Pruebas

No leer la tabla entera. Señalar cuatro: `-d` carpeta, `-f` archivo, `-z` vacía, `-ne` distinto (números). Y la última fila: `!` invierte cualquiera.

---

## Atajos

**Qué decir:** "`&&` es 'y si salió bien'; `||` es 'o si falló'. Son un `if` de una línea". El `>&2`: "los errores van por otro canal, para que no se mezclen con los datos". Lo van a ver con las manos en el lab, Parte 3.

---

## Que el script pare al primer error

**La frase que hay que decir:** *"sin `set -e`, un `cd /carpeta-que-no-existe` que falla no detiene nada, y la línea siguiente (un `rm`, por ejemplo) se ejecuta donde no debía. Con `set -e`, el script muere en el `cd`."* Se demuestra en el lab, Parte 4, con dos scripts de cuatro líneas.

---

## Revisar y depurar

**Qué decir:** "`bash -n` antes de ejecutar cualquier script nuevo. `bash -x` cuando no entienden qué hizo". Desde este bloque, cada script termina con `bash -n ... && echo "sintaxis ok"`.

---

## Dejar rastro

Los dos comandos, `logger` y `journalctl -t`.
**Qué señalar:** la línea del journal tiene fecha, servidor, la etiqueta `revisar` y el mensaje. "Cuando `backup.sh` corra solo de noche, así vamos a saber qué pasó."
