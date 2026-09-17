# Comandos — bucles y funciones (guía del instructor)

Diez minutos. Tres cosas que tienen que quedar: (1) `for x in lista; do …; done` hace lo mismo para cada elemento; (2) `while read -r a b c; do …; done < archivo` lee un archivo línea por línea y reparte cada línea en variables; (3) una función es un bloque con nombre que se llama como un comando. Los comandos de texto (`cut`, `sort`, `uniq`) son del Día 2. `tar` y `find -mtime` también.

**Qué es un bucle:** repetir un bloque de comandos varias veces, cambiando una variable en cada vuelta. Es la razón de escribir scripts: lo que hay que hacer con 40 archivos o 40 usuarios se escribe una vez.
**Qué es `for var in lista; do … done`:** `var` toma el primer valor de la lista, se ejecuta el bloque; toma el segundo, se ejecuta; y así hasta el último. `do` abre el bloque, `done` lo cierra.
**Qué es `{1..5}`:** Bash lo convierte en `1 2 3 4 5` antes de ejecutar. Se llama expansión de llaves. `{1..3}` es `1 2 3`.
**Qué es un comodín en el `for`:** `*.log` lo reemplaza Bash por la lista de archivos que terminan en `.log` (globbing, Día 2). El `for` recibe esa lista.
**Qué es `for u in $(comando)`:** la lista es la salida del comando, palabra por palabra.
**Qué es `while read -r … ; done < archivo`:** `read` lee **una línea** del archivo y la reparte en las variables que le sigan, separando por espacios. `while` repite hasta que no quedan líneas. El `< archivo` al final del `done` es de dónde se lee. `-r` = leer la línea tal cual, sin interpretar las barras invertidas; se pone siempre.
**Qué pasa con la última variable de `read`:** se queda con **todo lo que sobra** de la línea. `read -r fecha hora nivel resto`: si la línea tiene siete palabras, `resto` recibe las últimas cuatro juntas. Por eso el archivo de usuarios del Bloque 4 puede tener el nombre completo con espacios al final.
**Qué es `IFS=,`:** cambiar el separador de `read` de espacios a comas, solo para ese `read`. Sirve para archivos CSV (valores separados por comas). Hoy los archivos van separados por espacios y no hace falta.
**Qué es `grep -c`:** cuenta las líneas que contienen la palabra (*count*). Día 2.
**Qué es `cut -d' ' -f3`:** el campo 3 de cada línea, usando el espacio como separador. Día 2.
**Qué es `sort | uniq -c`:** ordena y después cuenta cuántas veces se repite cada línea. `uniq` solo cuenta repetidas consecutivas, por eso siempre va `sort` antes. Día 2.
**Qué es `wc -l < archivo`:** cuenta las líneas. Con `<` en vez de pasarle el nombre, imprime solo el número, sin el nombre del archivo al lado. Útil dentro de `$( )`.
**Qué es una función:** un bloque de comandos con nombre. `log() { …; }` la define; `log "mensaje"` la ejecuta. Adentro, `$1` es lo que le pasaron a la función (no al script). Se define arriba y se usa abajo.
**Qué es `${1:-valor}`:** "el primer argumento, y si no llegó ninguno, este valor". Es la forma de tener valores por defecto: `backup.sh` sin argumentos respalda `~/empresa`; con uno, respalda eso.
**Qué es `basename`:** la última parte de una ruta. `basename /home/student/empresa` → `empresa`.
**Qué es `dirname`:** todo menos la última parte. `dirname /home/student/empresa` → `/home/student`. Los dos juntos separan "dónde está" de "cómo se llama".
**Qué es `systemctl is-active`:** dice `active` o `inactive` para un servicio (Día 4). Sin `sudo`.
**Qué es `tar -czf ARCHIVO -C CARPETA NOMBRE`:** crear (`c`) comprimido con gzip (`z`) el archivo (`f`) `ARCHIVO`, parándose primero en `CARPETA` (`-C`) y guardando `NOMBRE`. Así adentro queda `empresa/...` y no `home/student/empresa/...`. Día 2.
**Qué es `tar -tzf`:** listar (`t`) qué hay adentro sin extraer. **`tar -xzf … -C destino`:** extraer (`x`) en esa carpeta. Día 2.
**Qué es `find … -mtime +7`:** archivos modificados hace **más** de 7 días (Día 2). **`-print -delete`:** mostrar cada uno y borrarlo. Con `| wc -l` al final, el script cuenta cuántos borró.
**Qué es la retención:** cuántos días se guardan los respaldos antes de borrarlos. Acá, 7.
**Qué es `touch -d "-10 days" archivo`:** crear el archivo (vacío) con fecha de modificación de hace 10 días. Sirve para simular respaldos viejos sin esperar 10 días.
**Qué es `ls -t`:** listar ordenado por fecha de modificación, el más nuevo primero. Con `| head -1`, el más nuevo.
**Qué es `date +%F-%H%M`:** la fecha como `2026-09-16-1058` (año-mes-día-horaminuto). Va en el nombre del respaldo. **`date '+%F %T'`:** `2026-09-16 10:58:10`, para las líneas de log.
**Qué es `mkdir -p`:** crea la carpeta y las intermedias; si ya existe, no dice nada. Día 2.

---

## `for`: lo mismo, para cada cosa de una lista

**Qué decir, leyendo el diagrama:** "`svc` vale `sshd`, se ejecuta el `echo`; `svc` vale `crond`, se ejecuta el `echo`; `svc` vale `chronyd`, otra vez. `done` cierra".
**Qué señalar:** la tabla: la lista puede ser palabras, números (`{1..5}`), archivos (`*.log`) o la salida de un comando. Todo `for` es igual; solo cambia de dónde sale la lista.

---

## `while read`: una línea por vez de un archivo

**La frase que hay que decir:** *"`read` agarra una línea y la reparte en las variables por los espacios. La última variable se lleva todo lo que sobra."* Es lo que hace que funcione el archivo de usuarios del Bloque 4.
**Qué señalar:** el `< app.log` pegado al `done`: ahí se dice qué archivo se lee.

---

## Contar cosas de un archivo

Son los comandos del Día 2. Una frase: "`grep -c` cuenta líneas con una palabra; `cut | sort | uniq -c` cuenta cuántas veces aparece cada valor de una columna".

---

## Funciones: un bloque con nombre

**Qué decir:** "cuando un pedazo se repite, se le pone nombre. `log` escribe en pantalla con la hora **y** en el log del sistema; en vez de escribir esas dos líneas cinco veces, escribimos `log "mensaje"`".
**Qué señalar:** adentro de la función, `$1` es el mensaje que le pasaron a `log`, no el argumento del script.

---

## Valores por defecto

Los dos comandos, `basename` y `dirname`.
**Qué decir:** "`${1:-/home/student/empresa}` se lee: 'el primer argumento, o si no vino, esta ruta'. `basename` da el nombre, `dirname` da dónde está".

---

## Lo que usa `backup.sh`

No demostrar acá; se hace en el Lab 2. Señalar dos filas: `tar -czf … -C /home/student empresa` (el `-C` es para que adentro quede `empresa/` y no la ruta completa) y `find … -mtime +7 -print -delete` (busca, muestra y borra: la retención en un solo comando).
