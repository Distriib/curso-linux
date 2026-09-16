# Día 07 — Automatización: Bash scripting, cron, at, systemd timers y automatización empresarial
> Al terminar el día, el participante escribe scripts de Bash con variables, condicionales, bucles y funciones que resuelven tareas reales de administración (respaldos, altas de usuarios, monitoreo), los programa con cron, at y systemd timers, y entiende dónde encajan Ansible y Cockpit en la administración a escala.

**Ficha técnica cubierta:** RH134 M1: Bash scripting, Variables, Loops, Automatización de tareas · RH134 M2: Cron, At, Automatización empresarial.
**Requisitos previos:** snapshot `dia06-fin` restaurado o VM sana; `/datos` montado (LV del Día 6) o, en su defecto, el directorio `/datos` se crea al inicio del día; grupos de PanamaTech del Día 3 (`sistemas`, `soporte`, `auditoria`) — si alguno falta, el script del Lab 3.1 lo crea; `httpd` instalado el Día 4 (opcional: `healthcheck.sh` lo detecta); acceso a los repositorios (`sudo dnf repolist` lista `baseos` y `appstream`; recuerde que con Simple Content Access `subscription-manager status` dice `Overall Status: Disabled` y eso es normal) porque se instalan `at`, `ansible-core` y, si falta, `cockpit`; acceso por `ssh -p 2222 student@localhost`.

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---|---|---|
| 0:00–0:05 | 5 min | Repaso | Tres preguntas del Día 6 con la terminal abierta; verificación de `/datos` y creación de `~/bin` |
| 0:05–0:55 | 50 min | Bloque 1 | Fundamentos de Bash scripting: shebang, `chmod +x`, PATH, variables, argumentos, sustitución de comandos, `test`/`[[ ]]`, `if`/`case`, exit codes, `set -euo pipefail`, `bash -x` |
| 0:55–1:40 | 45 min | Bloque 2 | Bucles (`for`, `while read`, `until`), funciones, arrays, procesamiento de texto (`cut`, `awk`, `sed`, `sort`, `uniq`, `tr`, `xargs`); `backup.sh` versión 2 |
| 1:40–2:05 | 25 min | Bloque 3 | Scripts de administración: `crear-usuarios.sh`, `healthcheck.sh`, `monitor-disco.sh` (2 min de conceptos + 3 labs) |
| 2:05–2:20 | 15 min | Descanso | |
| 2:20–3:10 | 50 min | Bloque 4 | Programación de tareas: cron y anacron, `at`, systemd timers, `systemd-tmpfiles` |
| 3:10–3:25 | 15 min | Bloque 5 | Automatización empresarial: infraestructura como código, Ansible (playbook mínimo), Cockpit, Satellite e Insights |
| 3:25–3:50 | 25 min | Reto | Ticket: `limpiar-logs.sh` programado con cron y con systemd timer |
| 3:50–4:00 | 10 min | Cierre | Resumen, cheatsheet, snapshot `dia07-fin`, tarea |

## Prioridad si falta tiempo

**Imprescindible**
- Escribir y ejecutar un script: shebang, `chmod +x`, `~/bin` en el PATH, variables, `$1`, `$#`, `$?`, `$( )`.
- `if`/`test` con `-d`, `-f`, `-z`, comparación de números y cadenas; `exit N` con códigos coherentes.
- `for` sobre una lista y sobre `*.log`; `while read -r` sobre un archivo.
- `backup.sh` versión 2 (Lab 2.2) funcionando y programado en el crontab del usuario.
- cron: `crontab -e/-l/-r`, los 5 campos, `PATH` y redirección de salida en el crontab, `/var/log/cron`.
- `at now + 5 minutes`, `atq`, `atrm`.
- systemd timer `monitor-disco.timer` con `OnCalendar` y `Persistent=true`, `systemctl list-timers`.

**Importante**
- `set -euo pipefail` y `bash -x`.
- Funciones, `case`, here-doc, `printf`.
- `crear-usuarios.sh` y `healthcheck.sh` (Labs 3.1 y 3.2).
- `awk '{print $N}'`, `sed -i`, `sort | uniq -c`.
- `/etc/crontab`, `/etc/cron.d/`, `cron.hourly|daily|weekly|monthly`, anacron, `cron.allow`/`cron.deny`.
- `systemd-tmpfiles` y `/etc/tmpfiles.d/`.
- Reto individual (puede quedar como tarea con la solución publicada al día siguiente).

**Si sobra tiempo**
- Arrays, `until`, `break`/`continue`, `batch`, `at -c`.
- Ansible: instalación, inventario y playbook (puede quedar como demo del instructor).
- Cockpit por `https://localhost:9090` (demo del instructor; requiere port forwarding 9090 o el túnel `ssh -p 2222 -L 9090:localhost:9090 -N student@localhost` del Día 5).
- Mención de Satellite, Insights y Ansible Automation Platform.

---

## Bloque 1 — Fundamentos de Bash scripting

### Conceptos (20 min)

**Qué decir en clase.** Un script es simplemente una lista de comandos que el administrador ya sabe escribir en la terminal, guardada en un archivo para no volver a teclearla. Todo lo que funciona en el prompt funciona en un script. La diferencia entre "saber Linux" y "administrar Linux" es, en buena parte, dejar de hacer las cosas a mano y empezar a dejarlas escritas: se documentan solas, se repiten igual cada vez y se pueden programar.

**Anatomía de un script**

- **Shebang** `#!/bin/bash` en la primera línea: le dice al kernel con qué intérprete ejecutar el archivo. Sin shebang, el sistema usa la shell actual y el resultado puede variar. En RHEL 9 `/bin` es un enlace a `/usr/bin`, así que `#!/bin/bash` y `#!/usr/bin/bash` son equivalentes.
- **Permiso de ejecución**: `chmod +x script.sh`. Un archivo de texto con shebang pero sin `x` no se puede ejecutar como `./script.sh`; sí con `bash script.sh` (ahí el shebang se ignora, porque el intérprete ya se indicó).
- **Tres formas de ejecutar**: `./script.sh` (ruta relativa, necesita `x`), `bash script.sh` (no necesita `x`, útil para probar) y `script.sh` a secas si el directorio está en el `PATH`. Convención: scripts personales en `~/bin` (RHEL 9 lo incluye en el PATH del usuario desde `~/.bashrc`), scripts del sistema en `/usr/local/bin` (para que los ejecute root, cron o systemd).
- **Comentarios** con `#`. La cabecera de todo script debe decir qué hace, cómo se usa y qué códigos de salida devuelve: quien lo lea dentro de un año será otro administrador… o el mismo, sin memoria.

**Variables**

- Asignación **sin espacios**: `NOMBRE=valor`. `NOMBRE = valor` intenta ejecutar un comando llamado `NOMBRE`.
- Lectura con `$NOMBRE` o `${NOMBRE}`; las llaves son obligatorias cuando el texto sigue pegado: `${NOMBRE}_backup.tar.gz`.
- **Comillas dobles** expanden variables y `$( )`: `"Hola $USER"`. **Comillas simples** no expanden nada: `'Hola $USER'` imprime literalmente `$USER`. Regla de oro: **siempre entre comillas dobles las variables que contienen rutas o texto** (`"$RUTA"`), para que un espacio en el nombre no rompa el comando.
- `readonly VERSION=1.0` impide reasignar. Por convención, MAYÚSCULAS para constantes y variables de entorno, minúsculas para variables locales del script.
- **Variables especiales**: `$0` nombre del script, `$1 … $9` argumentos, `$#` cantidad de argumentos, `$@` todos los argumentos (usar `"$@"`), `$?` código de salida del último comando, `$$` PID del proceso actual.
- **Sustitución de comandos** `$(comando)`: `FECHA=$(date +%F)`. La forma con acentos graves `` `comando` `` es antigua; no la use.
- **Aritmética** `$(( ))`: `TOTAL=$((5 + 3 * 2))` → 11. Solo enteros: `$((10 / 3))` → 3.
- `read -p "Texto: " variable` lee del teclado; `read -s` oculta lo escrito (contraseñas). En scripts programados con cron **no hay teclado**: todo debe venir por argumentos o archivos.

**Códigos de salida y pruebas**

- Todo comando termina con un código: **0 = éxito, distinto de 0 = fallo**. `echo $?` lo muestra. `exit N` fija el código del script. Convención mínima: `exit 0` ok, `exit 1` error de uso o de entrada, `exit 2` en adelante errores específicos. Cron, systemd y Ansible se fijan en ese código.
- `test`, `[ ]` y `[[ ]]` evalúan condiciones. `[ ]` es el `test` clásico (POSIX, exige un espacio después de `[` y otro antes de `]`, y comillas en las variables); `[[ ]]` es propio de Bash, más tolerante y con más operadores. En scripts de Bash use `[[ ]]`.
- Archivos: `-e` existe, `-f` es archivo regular, `-d` es directorio, `-r` legible, `-w` escribible, `-x` ejecutable, `-s` no está vacío.
- Cadenas: `-z` vacía, `-n` no vacía, `==`/`!=` igual/distinto.
- Números: `-eq -ne -lt -le -gt -ge`. Ojo: `[[ 10 > 9 ]]` compara como texto ("1" < "9") y da falso; para números use `-gt` o `(( 10 > 9 ))`.
- Estructuras: `if … then … elif … then … else … fi`; `case "$var" in patron) …;; *) …;; esac` para menús y opciones.
- Operadores de control: `a && b` ejecuta `b` solo si `a` tuvo éxito; `a || b` solo si `a` falló; `a; b` ejecuta ambos. Muy usado: `[[ -d "$DIR" ]] || mkdir -p "$DIR"`.

**Salida con formato**

- `echo -e "Línea 1\nLínea 2\tcon tabulador"` interpreta secuencias.
- `printf "%-20s %8s\n" "Nombre" "Tamaño"` alinea columnas (`%-20s` texto a la izquierda en 20 caracteres, `%8s` a la derecha en 8, `%d` entero).
- **Here-doc**: `cat <<FIN … FIN` escribe varias líneas de una vez, con variables expandidas; `<<'FIN'` (delimitador entre comillas) las deja literales. Útil para generar archivos de configuración desde un script.

**Modo estricto y depuración**

- `set -euo pipefail` al inicio del script: `-e` termina el script si un comando falla, `-u` trata una variable no definida como error, `-o pipefail` hace que una tubería falle si falla cualquiera de sus etapas (no solo la última). Sin esto, un `cd /ruta/inexistente && rm -rf *` que falla en el `cd` sigue adelante… Con `-e`, se detiene. Advertencia: `-e` no actúa dentro de un `if` ni con `&&`/`||`, y `((i++))` devuelve 1 cuando `i` vale 0 (use `i=$((i+1))`).
- `bash -n script.sh` revisa la sintaxis sin ejecutar. `bash -x script.sh` muestra cada comando expandido con `+` delante: es la herramienta de depuración número uno.
- `logger -t etiqueta "mensaje"` escribe en el journal y en `/var/log/messages`: es la forma correcta de que un script deje rastro cuando lo ejecuta cron a las 3 de la mañana.

**Buenas prácticas que evitan tickets**
1. Comillas dobles en todas las variables con rutas.
2. Rutas absolutas dentro de scripts que va a ejecutar cron (`/usr/bin/tar`, no `tar`; o definir `PATH` al inicio).
3. Validar argumentos y directorios antes de actuar; salir con mensaje y código distinto de 0.
4. Registrar con `logger` y con `echo` con fecha (`$(date '+%F %T')`).
5. Nunca editar scripts en el Bloc de notas de Windows: los finales de línea CRLF producen el error `/bin/bash^M: bad interpreter`.

### Lab 1.1 — Primer script, variables y argumentos (15 min)

- **Objetivo:** crear `~/bin`, escribir tres scripts cortos y ejecutarlos por nombre desde cualquier directorio.

1. Preparar el entorno del día (todos ejecutan esto, esté o no montado `/datos`):

```bash
findmnt /datos || echo "/datos no está montado: se usará como directorio normal dentro de /"
sudo mkdir -p /datos/backups /datos/tmp
sudo chown student:student /datos/backups
mkdir -p ~/bin ~/empresa/logs ~/empresa/documentos
echo "$PATH" | tr ':' '\n' | grep bin
```

Salida esperada (entre otras líneas):
```
/home/student/.local/bin
/home/student/bin
/usr/local/bin
/usr/bin
```
Qué observar: `findmnt /datos` muestra una línea con `/dev/mapper/vg_datos-lv_datos` si el LV del Día 6 está montado (`df -h /datos` no sirve para comprobarlo: siempre responde, aunque sea con los datos de `/`). `~/bin` ya está en el PATH aunque no existiera; lo agrega `~/.bashrc` en RHEL 9. Si no aparece, agregue `export PATH="$HOME/bin:$PATH"` al final de `~/.bashrc` y ejecute `source ~/.bashrc`. Quien tenga montado `/backups` (reto del Día 6) puede usarlo como destino de los respaldos pasándolo como segundo argumento a `backup.sh` en el Lab 2.2; el material usa `/datos/backups` para que todos tengan la misma ruta.

2. Primer script:

```bash
cat > ~/bin/hola.sh <<'EOF'
#!/bin/bash
# hola.sh - primer script: variables, comillas y sustitución de comandos
NOMBRE="Procuraduría"
FECHA=$(date +%F)
readonly VERSION=1.0

echo "Hola, $NOMBRE. Hoy es $FECHA (script v$VERSION)"
echo 'Con comillas simples no se expande: $NOMBRE'
echo "Archivo de salida: ${NOMBRE}_reporte.txt"
echo "Operación aritmética: $((7 * 6))"
EOF
ls -l ~/bin/hola.sh
./hola.sh 2>/dev/null || echo "sin ruta ni permiso aún"
bash ~/bin/hola.sh
```

Salida esperada:
```
-rw-r--r--. 1 student student 345 Sep  3 08:12 /home/student/bin/hola.sh
sin ruta ni permiso aún
Hola, Procuraduría. Hoy es 2026-09-03 (script v1.0)
Con comillas simples no se expande: $NOMBRE
Archivo de salida: Procuraduría_reporte.txt
Operación aritmética: 42
```
Qué observar: `bash hola.sh` funciona sin permiso de ejecución; `./hola.sh` no funciona ni por ruta (está en `~/bin`, no en el directorio actual) ni por permiso.

3. Dar permiso y ejecutar por nombre desde cualquier lugar:

```bash
chmod +x ~/bin/hola.sh
cd /tmp && hola.sh && cd ~
```

Salida esperada: las mismas cuatro líneas del paso anterior. Qué observar: el script se encuentra porque `~/bin` está en el PATH; no hace falta `./` ni ruta completa.

4. Variables especiales y argumentos:

```bash
cat > ~/bin/args.sh <<'EOF'
#!/bin/bash
# args.sh - muestra las variables especiales de Bash
echo "Nombre del script (\$0): $0"
echo "Primer argumento  (\$1): $1"
echo "Segundo argumento (\$2): $2"
echo "Cantidad          (\$#): $#"
echo "Todos             (\$@): $@"
echo "PID del script    (\$\$): $$"
ls /noexiste 2>/dev/null
echo "Código de salida de ls (\$?): $?"
true
echo "Código de salida de true (\$?): $?"
EOF
chmod +x ~/bin/args.sh
args.sh servidor01 "Sala de servidores" 42
```

Salida esperada:
```
Nombre del script ($0): /home/student/bin/args.sh
Primer argumento  ($1): servidor01
Segundo argumento ($2): Sala de servidores
Cantidad          ($#): 3
Todos             ($@): servidor01 Sala de servidores 42
PID del script    ($$): 4127
Código de salida de ls ($?): 2
Código de salida de true ($?): 0
```
Qué observar: las comillas dobles mantienen "Sala de servidores" como un solo argumento (por eso `$#` es 3 y no 5). `ls` sobre algo inexistente devuelve 2.

5. Here-doc, `read -p` y `printf` en un reporte:

```bash
cat > ~/bin/reporte.sh <<'EOF'
#!/bin/bash
# reporte.sh - reporte breve del servidor usando here-doc y printf
read -p "Nombre del técnico: " TECNICO
cat <<FIN
=== Reporte de $(hostname) ===
Generado por: $TECNICO
Fecha:        $(date '+%F %T')
Kernel:       $(uname -r)
Uptime:       $(uptime -p)
FIN
printf "%-12s %8s %8s\n" "Filesystem" "Tamaño" "Uso"
printf "%-12s %8s %8s\n" "/" "$(df -h / | awk 'NR==2 {print $2}')" "$(df -h / | awk 'NR==2 {print $5}')"
EOF
chmod +x ~/bin/reporte.sh
reporte.sh
```

Salida esperada (escribir un nombre cuando lo pida):
```
Nombre del técnico: Esteban
=== Reporte de rhel01 ===
Generado por: Esteban
Fecha:        2026-09-03 08:20:41
Kernel:       5.14.0-570.el9.x86_64
Uptime:       up 1 hour, 12 minutes
Filesystem     Tamaño      Uso
/                 17G      18%
```
Qué observar: dentro del here-doc se expanden `$TECNICO` y `$(…)`. La versión del kernel depende de la 9.x instalada; en UTM termina en `aarch64`.

- **Checkpoint:** pegar en el chat la salida de:
```bash
args.sh uno dos | head -4
```

### Lab 1.2 — Condicionales, exit codes, modo estricto y depuración (15 min)

- **Objetivo:** escribir un script que valida argumentos, distingue archivo/directorio y devuelve códigos de salida distintos; ver el efecto de `set -euo pipefail` y de `bash -x`.

1. Script con `if`/`elif`/`else` y validación de argumentos:

```bash
cat > ~/bin/revisar.sh <<'EOF'
#!/bin/bash
# revisar.sh - informa qué es una ruta (archivo o directorio)
# Uso: revisar.sh RUTA
# Salida: 0 ok, 1 uso incorrecto, 2 la ruta no existe
if [[ $# -ne 1 ]]; then
    echo "Uso: $0 RUTA" >&2
    exit 1
fi

RUTA="$1"

if [[ -d "$RUTA" ]]; then
    echo "$RUTA es un directorio con $(ls -A "$RUTA" | wc -l) elementos"
elif [[ -f "$RUTA" ]]; then
    echo "$RUTA es un archivo de $(stat -c %s "$RUTA") bytes"
    [[ -x "$RUTA" ]] && echo "  además es ejecutable"
    [[ -w "$RUTA" ]] || echo "  no tiene permiso de escritura para $USER"
else
    echo "ERROR: $RUTA no existe" >&2
    exit 2
fi
EOF
chmod +x ~/bin/revisar.sh
revisar.sh /etc
revisar.sh /etc/hostname
revisar.sh ~/bin/hola.sh
revisar.sh /nada; echo "código: $?"
revisar.sh; echo "código: $?"
```

Salida esperada:
```
/etc es un directorio con 176 elementos
/etc/hostname es un archivo de 7 bytes
  no tiene permiso de escritura para student
/home/student/bin/hola.sh es un archivo de 345 bytes
  además es ejecutable
ERROR: /nada no existe
código: 2
Uso: /home/student/bin/revisar.sh RUTA
código: 1
```
Qué observar: los mensajes de error van a `>&2` (stderr) para que no se mezclen con la salida normal cuando el script se usa en una tubería. El número de elementos de `/etc` varía.

2. `case` para opciones tipo menú:

```bash
cat > ~/bin/nivel.sh <<'EOF'
#!/bin/bash
# nivel.sh - clasifica un porcentaje de uso de disco
USO="${1:-}"
if [[ -z "$USO" ]]; then
    echo "Uso: $0 PORCENTAJE" >&2; exit 1
fi
case "$USO" in
    [0-9]|[1-6][0-9])  echo "$USO% -> normal" ;;
    7[0-9]|8[0-9])     echo "$USO% -> advertencia" ;;
    9[0-9]|100)        echo "$USO% -> crítico"; exit 3 ;;
    *)                 echo "$USO no es un porcentaje válido" >&2; exit 1 ;;
esac
EOF
chmod +x ~/bin/nivel.sh
nivel.sh 35; nivel.sh 82; nivel.sh 97; echo "código: $?"; nivel.sh abc; echo "código: $?"
```

Salida esperada:
```
35% -> normal
82% -> advertencia
97% -> crítico
código: 3
abc no es un porcentaje válido
código: 1
```

3. Efecto de `set -euo pipefail`:

```bash
cat > /tmp/sin-set.sh <<'EOF'
#!/bin/bash
echo "Antes del error"
cat /noexiste
echo "Después del error (se ejecuta igual)"
echo "Variable sin definir: [$NO_DEFINIDA]"
EOF
cat > /tmp/con-set.sh <<'EOF'
#!/bin/bash
set -euo pipefail
echo "Antes del error"
cat /noexiste
echo "Después del error (NO se ejecuta)"
EOF
bash /tmp/sin-set.sh; echo "código: $?"
echo ----
bash /tmp/con-set.sh; echo "código: $?"
```

Salida esperada:
```
Antes del error
cat: /noexiste: No such file or directory
Después del error (se ejecuta igual)
Variable sin definir: []
código: 0
----
Antes del error
cat: /noexiste: No such file or directory
código: 1
```
Qué observar: sin `set -e` el script "termina bien" (código 0) aunque falló en medio: cron nunca se enteraría. Con `set -e` se detiene en el primer error y devuelve el código del comando que falló.

4. `pipefail` y `-u` por separado:

```bash
bash -c 'cat /noexiste | wc -l; echo "código sin pipefail: $?"'
bash -c 'set -o pipefail; cat /noexiste | wc -l; echo "código con pipefail: $?"'
bash -c 'set -u; echo "$VARIABLE_QUE_NO_EXISTE"'; echo "código: $?"
```

Salida esperada:
```
cat: /noexiste: No such file or directory
0
código sin pipefail: 0
cat: /noexiste: No such file or directory
0
código con pipefail: 1
bash: line 1: VARIABLE_QUE_NO_EXISTE: unbound variable
código: 1
```
Qué observar: `wc -l` imprime `0` y devuelve 0, así que sin `pipefail` la tubería "tuvo éxito" aunque `cat` falló. Con `set -u`, Bash aborta con código 1 en cuanto lee una variable sin definir.

5. Depurar con `bash -n` y `bash -x`:

```bash
bash -n ~/bin/revisar.sh && echo "sintaxis correcta"
bash -x ~/bin/revisar.sh /etc/hostname
```

Salida esperada (resumida):
```
sintaxis correcta
+ [[ 1 -ne 1 ]]
+ RUTA=/etc/hostname
+ [[ -d /etc/hostname ]]
+ [[ -f /etc/hostname ]]
++ stat -c %s /etc/hostname
+ echo '/etc/hostname es un archivo de 7 bytes'
/etc/hostname es un archivo de 7 bytes
+ [[ -x /etc/hostname ]]
+ [[ -w /etc/hostname ]]
+ echo '  no tiene permiso de escritura para student'
  no tiene permiso de escritura para student
```
Qué observar: cada `+` es un comando ya expandido; `++` es un comando dentro de `$( )`. Cuando un script "hace algo raro", `bash -x` muestra exactamente qué valor tenía cada variable.

6. Dejar rastro con `logger`:

```bash
logger -t revisar "Prueba de logger desde el Lab 1.2 por $USER"
sudo journalctl -t revisar -n 1 --no-pager
sudo grep revisar /var/log/messages | tail -1
```

Salida esperada:
```
Sep 03 08:31:07 rhel01 revisar[4410]: Prueba de logger desde el Lab 1.2 por student
Sep  3 08:31:07 rhel01 revisar[4410]: Prueba de logger desde el Lab 1.2 por student
```

- **Checkpoint:** pegar en el chat la salida de:
```bash
revisar.sh /nada; echo "código: $?"; nivel.sh 95; echo "código: $?"
```

---

## Bloque 2 — Bucles, funciones y procesamiento de texto

### Conceptos (15 min)

**Bucles: la razón de ser de los scripts.** Si hay que hacer lo mismo con 40 archivos o 40 usuarios, se escribe una vez dentro de un bucle.

- `for var in lista; do …; done` recorre palabras: `for svc in sshd crond chronyd`.
- `for f in *.log` recorre archivos (la shell expande el comodín; si no hay coincidencias, la variable recibe literalmente `*.log`: comprobar con `[[ -e "$f" ]]`).
- `for i in {1..10}` genera secuencias; `for (( i=1; i<=10; i++ ))` es la forma tipo C.
- `while read -r linea; do …; done < archivo` procesa un archivo línea a línea. `-r` evita que las barras invertidas se interpreten. Con `IFS=,` antes de `read` se separan campos por coma: `while IFS=, read -r usuario grupo comentario`.
- `until condición; do …; done` repite **hasta** que la condición sea verdadera (por ejemplo, esperar a que un servicio responda).
- `break` sale del bucle; `continue` salta a la siguiente iteración.
- Trampa clásica: `comando | while read …; do total=$((total+1)); done` ejecuta el `while` en una subshell y **las variables modificadas se pierden**. Solución: `while read …; do …; done < <(comando)` (sustitución de proceso).

**Funciones.** `nombre() { …; }`. Reciben argumentos como un script (`$1`, `$#`), `local` declara variables locales y `return N` fija el código de salida (0–255), no un valor: para devolver texto se usa `echo` y se captura con `$(nombre)`. Regla: si un bloque se repite dos veces, va en una función; si el script pasa de 40 líneas, se divide en funciones con nombres claros (`validar_args`, `hacer_backup`, `log`).

**Arrays básicos.** `servicios=(sshd crond chronyd)`; `${servicios[0]}` primer elemento; `"${servicios[@]}"` todos; `${#servicios[@]}` cantidad; `servicios+=(httpd)` agrega. Sirven para listas que el script recorre o comprueba.

**Herramientas de texto que un administrador usa a diario** (solo lo esencial):

| Herramienta | Uso típico | Ejemplo |
|---|---|---|
| `cut` | cortar columnas por delimitador fijo | `cut -d: -f1 /etc/passwd` |
| `awk` | columnas por espacios, filtros y cálculos | `df -h \| awk 'NR>1 {print $6, $5}'` · `awk -F: '$3 >= 1000 {print $1}' /etc/passwd` |
| `sed` | sustituir/borrar líneas, editar en sitio | `sed -i.bak 's/DEBUG/INFO/g' app.log` · `sed -n '5,10p' archivo` · `sed '/^#/d' archivo` |
| `grep` | filtrar líneas | `grep -c ERROR app.log` · `grep -v INFO` |
| `sort` / `uniq -c` | contar repeticiones (siempre `sort` antes de `uniq`) | `awk '{print $3}' app.log \| sort \| uniq -c \| sort -rn` |
| `tr` | traducir/borrar caracteres | `tr 'a-z' 'A-Z'` · `tr -d '\r'` |
| `xargs` | convertir una lista en argumentos | `find . -name "*.log" -print0 \| xargs -0 wc -l` |

En `awk`, `$1…$N` son las columnas (separadas por espacios o por `-F`), `NR` es el número de línea, `{print}` imprime. Con eso se resuelve el 90 % de lo que un administrador necesita. `sed -i` modifica el archivo original: use `-i.bak` la primera vez.

### Lab 2.1 — Bucles, funciones y texto sobre un log de aplicación (15 min)

- **Objetivo:** generar un log de prueba idéntico para todos y procesarlo con bucles, `awk`, `sed`, `sort`/`uniq` y una función.

1. Generar el log (60 líneas deterministas):

```bash
mkdir -p ~/empresa/logs
: > ~/empresa/logs/app.log
for i in $(seq 0 59); do
    if (( i % 7 == 0 )); then nivel=ERROR; elif (( i % 5 == 0 )); then nivel=WARN; else nivel=INFO; fi
    case $(( i % 3 )) in 0) usuario=ana ;; 1) usuario=carlos ;; 2) usuario=pedro ;; esac
    printf '2026-09-01 08:%02d:00 %s usuario=%s servidor=web%02d accion=login\n' \
        "$i" "$nivel" "$usuario" $(( i % 2 + 1 )) >> ~/empresa/logs/app.log
done
wc -l ~/empresa/logs/app.log
head -3 ~/empresa/logs/app.log
```

Salida esperada:
```
60 /home/student/empresa/logs/app.log
2026-09-01 08:00:00 ERROR usuario=ana servidor=web01 accion=login
2026-09-01 08:01:00 INFO usuario=carlos servidor=web02 accion=login
2026-09-01 08:02:00 INFO usuario=pedro servidor=web01 accion=login
```
Qué observar: el propio generador usa `for`, `if`, `case`, aritmética y `printf`.

2. Contar por nivel y por usuario con `awk`, `sort` y `uniq`:

```bash
awk '{print $3}' ~/empresa/logs/app.log | sort | uniq -c | sort -rn
awk '{print $4}' ~/empresa/logs/app.log | cut -d= -f2 | sort | uniq -c
grep -c ERROR ~/empresa/logs/app.log
awk '$3 == "ERROR" {print $2, $4}' ~/empresa/logs/app.log | head -3
```

Salida esperada:
```
     41 INFO
     10 WARN
      9 ERROR
     20 ana
     20 carlos
     20 pedro
9
08:00:00 usuario=ana
08:07:00 usuario=carlos
08:14:00 usuario=pedro
```

3. `sed` para sustituir en sitio con respaldo, y `tr`:

```bash
sed -i.bak 's/WARN/ADVERTENCIA/g' ~/empresa/logs/app.log
grep -c ADVERTENCIA ~/empresa/logs/app.log
ls ~/empresa/logs/
sed -n '1,2p' ~/empresa/logs/app.log | tr 'a-z' 'A-Z'
mv ~/empresa/logs/app.log.bak ~/empresa/logs/app.log
```

Salida esperada (quien conserve los archivos del Día 2 verá también `errores.txt` y `servidor.log`):
```
10
app.log  app.log.bak  errores.txt  servidor.log
2026-09-01 08:00:00 ERROR USUARIO=ANA SERVIDOR=WEB01 ACCION=LOGIN
2026-09-01 08:01:00 INFO USUARIO=CARLOS SERVIDOR=WEB02 ACCION=LOGIN
```

4. Bucles `for` de las cuatro formas y `while read`:

```bash
for svc in sshd crond chronyd; do printf "%-10s %s\n" "$svc" "$(systemctl is-active $svc)"; done
for f in ~/empresa/logs/*.log; do echo "$f: $(wc -l < "$f") líneas"; done
for i in {1..3}; do echo "web0$i"; done
for (( i=10; i>=8; i-- )); do echo "cuenta regresiva $i"; done
grep ERROR ~/empresa/logs/app.log | while read -r fecha hora nivel usuario resto; do echo "$hora -> $usuario"; done | head -2
```

Salida esperada:
```
sshd       active
crond      active
chronyd    active
/home/student/empresa/logs/app.log: 60 líneas
/home/student/empresa/logs/servidor.log: 300 líneas
web01
web02
web03
cuenta regresiva 10
cuenta regresiva 9
cuenta regresiva 8
08:00:00 -> usuario=ana
08:07:00 -> usuario=carlos
```
Qué observar: el segundo bucle imprime también `servidor.log` (los 300 renglones que se generaron el Día 2), porque el comodín recorre **todos** los `.log` que existan; quien restauró un snapshot limpio verá solo `app.log`. `errores.txt` no aparece: no termina en `.log`.

5. La trampa de la subshell, `until`, `break`/`continue` y una función. (Este bloque es largo: el instructor lo pega en el chat y los participantes lo copian tal cual; lo importante es leer el resultado, no teclearlo.)

```bash
cat > /tmp/bucles.sh <<'EOF'
#!/bin/bash
# bucles.sh - subshell vs sustitución de proceso, until, break/continue, funciones
LOG=/home/student/empresa/logs/app.log

total=0
grep ERROR "$LOG" | while read -r linea; do total=$((total+1)); done
echo "Errores contados con pipe (se pierde):   $total"

total=0
while read -r linea; do total=$((total+1)); done < <(grep ERROR "$LOG")
echo "Errores contados con < <( ) (correcto):  $total"

n=1
until [[ $n -gt 3 ]]; do echo "intento $n"; n=$((n+1)); done

for i in {1..10}; do
    [[ $i -eq 3 ]] && continue
    [[ $i -eq 6 ]] && break
    echo -n "$i "
done
echo

contar_nivel() {
    local nivel="$1"
    grep -c " $nivel " "$LOG"
}
es_root() { [[ $EUID -eq 0 ]]; }

for nivel in INFO WARN ERROR; do
    echo "$nivel: $(contar_nivel "$nivel")"
done
if es_root; then echo "Ejecutado como root"; else echo "Ejecutado como usuario normal ($USER)"; fi

servicios=(sshd crond chronyd)
servicios+=(firewalld)
echo "Hay ${#servicios[@]} servicios; el primero es ${servicios[0]}"
for s in "${servicios[@]}"; do echo -n "$s=$(systemctl is-active "$s") "; done; echo
EOF
bash /tmp/bucles.sh
```

Salida esperada:
```
Errores contados con pipe (se pierde):   0
Errores contados con < <( ) (correcto):  9
intento 1
intento 2
intento 3
1 2 4 5
INFO: 41
WARN: 10
ERROR: 9
Ejecutado como usuario normal (student)
Hay 4 servicios; el primero es sshd
sshd=active crond=active chronyd=active firewalld=active
```
Qué observar: la primera cuenta da 0 porque el `while` corrió en una subshell. Esto explica muchos scripts "que no suman".

- **Checkpoint:** pegar en el chat la salida de:
```bash
awk '{print $3}' ~/empresa/logs/app.log | sort | uniq -c | sort -rn
```

### Lab 2.2 — backup.sh versión 2 (15 min)

- **Objetivo:** convertir el `backup.sh` del curso base en un script robusto: origen por argumento con valor por defecto, validaciones, `tar`, retención con `find -mtime +7 -delete`, registro con `logger` y códigos de salida.

1. Escribir el script en `~/bin`:

```bash
cat > ~/bin/backup.sh <<'EOF'
#!/bin/bash
# backup.sh - respalda un directorio en formato tar.gz y aplica retención
# Uso:    backup.sh [ORIGEN] [DESTINO]
#         ORIGEN  por defecto /home/student/empresa
#         DESTINO por defecto /datos/backups
# Salida: 0 ok · 1 el origen no existe · 2 fallo de tar
set -euo pipefail

ORIGEN="${1:-/home/student/empresa}"
DESTINO="${2:-/datos/backups}"
RETENCION_DIAS=7
FECHA=$(date +%F-%H%M)
NOMBRE=$(basename "$ORIGEN")
ARCHIVO="$DESTINO/${NOMBRE}-${FECHA}.tar.gz"

log() {
    echo "$(date '+%F %T') $*"
    logger -t backup "$*"
}

if [[ ! -d "$ORIGEN" ]]; then
    log "ERROR: el origen $ORIGEN no existe"
    exit 1
fi

mkdir -p "$DESTINO"

if tar -czf "$ARCHIVO" -C "$(dirname "$ORIGEN")" "$NOMBRE"; then
    log "OK: creado $ARCHIVO ($(du -h "$ARCHIVO" | cut -f1))"
else
    log "ERROR: tar falló al crear $ARCHIVO"
    exit 2
fi

BORRADOS=$(find "$DESTINO" -name "${NOMBRE}-*.tar.gz" -mtime +"$RETENCION_DIAS" -print -delete | wc -l)
log "Retención: $BORRADOS respaldo(s) de más de $RETENCION_DIAS días eliminado(s)"
EOF
chmod +x ~/bin/backup.sh
bash -n ~/bin/backup.sh && echo "sintaxis ok"
```

Qué observar: `tar -C directorio_padre nombre` guarda rutas relativas (`empresa/...`) y evita el aviso `Removing leading '/'`. `-mtime +7` significa "modificado hace más de 7 días completos" (en la práctica, 8 días o más).

2. Ejecutar con valores por defecto y con un origen inexistente:

```bash
backup.sh
echo "código: $?"
backup.sh /directorio/inexistente; echo "código: $?"
ls -lh /datos/backups/
```

Salida esperada (el tamaño depende de lo que tenga `~/empresa`: con `servidor.log` del Día 2 ronda los 4–8 K):
```
2026-09-03 08:58:10 OK: creado /datos/backups/empresa-2026-09-03-0858.tar.gz (8.0K)
2026-09-03 08:58:10 Retención: 0 respaldo(s) de más de 7 días eliminado(s)
código: 0
2026-09-03 08:58:21 ERROR: el origen /directorio/inexistente no existe
código: 1
total 8.0K
-rw-r--r--. 1 student student 4.3K Sep  3 08:58 empresa-2026-09-03-0858.tar.gz
```
Qué observar: `du -h` (lo que imprime el script) reporta el espacio **ocupado en disco**, redondeado al bloque; `ls -lh` reporta el tamaño real del archivo. Por eso los dos números no coinciden.

3. Simular respaldos viejos y comprobar la retención; verificar el contenido y restaurar en `/tmp`:

```bash
touch -d "-10 days" /datos/backups/empresa-2026-08-24-2330.tar.gz
touch -d "-3 days" /datos/backups/empresa-2026-08-31-2330.tar.gz
backup.sh
ls /datos/backups/
ULTIMO=$(ls -t /datos/backups/empresa-*.tar.gz | head -1); echo "$ULTIMO"
tar -tzf "$ULTIMO" | head -3
mkdir -p /tmp/restauracion && tar -xzf "$ULTIMO" -C /tmp/restauracion && ls /tmp/restauracion/empresa
```

Salida esperada:
```
2026-09-03 09:01:02 OK: creado /datos/backups/empresa-2026-09-03-0901.tar.gz (8.0K)
2026-09-03 09:01:02 Retención: 1 respaldo(s) de más de 7 días eliminado(s)
empresa-2026-08-31-2330.tar.gz  empresa-2026-09-03-0858.tar.gz  empresa-2026-09-03-0901.tar.gz
/datos/backups/empresa-2026-09-03-0901.tar.gz
empresa/
empresa/logs/
empresa/logs/app.log
backups  clientes  documentos  genera-log.sh  logs
```
Qué observar: se borró el de 10 días y quedó el de 3 días. `ls -t` ordena por fecha de modificación, así que `ULTIMO` es el respaldo recién creado (los archivos "viejos" creados con `touch -d` son vacíos y no son tar válidos: por eso no se usa un comodín directamente con `tar -tzf`, que con dos archivos interpretaría el segundo como nombre de miembro). El orden de las entradas dentro del tar y el contenido de `~/empresa` dependen de lo que se creó en los días anteriores (PanamaTech del Día 2).

4. Verificar el rastro en el journal:

```bash
sudo journalctl -t backup --no-pager -n 2
```

Salida esperada:
```
Sep 03 09:01:02 rhel01 backup[5210]: OK: creado /datos/backups/empresa-2026-09-03-0901.tar.gz (8.0K)
Sep 03 09:01:02 rhel01 backup[5212]: Retención: 1 respaldo(s) de más de 7 días eliminado(s)
```

- **Checkpoint:** pegar en el chat la salida de:
```bash
ls /datos/backups/ && sudo journalctl -t backup -n 1 --no-pager
```

---

## Bloque 3 — Scripts de administración

### Conceptos (2 min)

**Qué decir en clase.** Tres scripts que resuelven tareas reales de la Dirección de Informática: altas de usuarios en lote, revisión de salud del servidor y alerta de disco. Ninguno trae nada nuevo: son las piezas del Bloque 1 y del Bloque 2 puestas juntas (validaciones, `while read`, funciones, arrays, `awk`, `logger`, códigos de salida). Por eso **no se teclean**: el instructor pega cada bloque `cat > … <<'EOF'` en el chat, los participantes lo copian, lo ejecutan y lo leen en voz alta identificando qué parte hace qué. La meta del bloque no es escribir 150 líneas en 25 minutos, es saber leer un script ajeno y predecir su código de salida.

### Lab 3.1 — crear-usuarios.sh: altas en lote desde un CSV (10 min)

- **Objetivo:** leer `usuarios.csv` (usuario,grupo,comentario) con `while read`, crear el grupo si no existe, crear el usuario con contraseña inicial y forzar el cambio en el primer inicio de sesión con `chage -d 0`. Es la evolución del script de altas del Día 3: ahora lee un archivo externo y es idempotente.

1. Crear el CSV (la primera línea es un comentario que el script salta). Los grupos `soporte` y `auditoria` existen desde el Día 3; `desarrollo` es nuevo y el script debe crearlo:

```bash
cat > ~/usuarios.csv <<'EOF'
# usuario,grupo,comentario
lmartinez,desarrollo,Luis Martinez - Desarrollo
rgomez,desarrollo,Rosa Gomez - Desarrollo
jperez,soporte,Juan Perez - Soporte
mcastro,auditoria,Maria Castro - Auditoria
EOF
cat -A ~/usuarios.csv | head -2
getent group sistemas soporte auditoria desarrollo
```

Salida esperada (`$` al final de cada línea y **sin** `^M`; si aparece `^M$` el archivo tiene finales CRLF: `sed -i 's/\r$//' ~/usuarios.csv`; `getent` no muestra `desarrollo` porque aún no existe):
```
# usuario,grupo,comentario$
lmartinez,desarrollo,Luis Martinez - Desarrollo$
sistemas:x:3001:ana,carlos
soporte:x:3002:pedro
auditoria:x:3003:laura
```

2. El script:

```bash
cat > ~/bin/crear-usuarios.sh <<'EOF'
#!/bin/bash
# crear-usuarios.sh - crea usuarios en lote a partir de un CSV
# Formato del CSV: usuario,grupo,comentario   (líneas con # se ignoran)
# Uso:    sudo /home/student/bin/crear-usuarios.sh /ruta/usuarios.csv
# Salida: 0 ok · 1 error de uso o de permisos
set -euo pipefail

CSV="${1:-}"
CLAVE_INICIAL="Cambiar.2026"

if [[ $EUID -ne 0 ]]; then
    echo "ERROR: ejecutar con sudo (crea usuarios)" >&2
    exit 1
fi
if [[ -z "$CSV" || ! -r "$CSV" ]]; then
    echo "Uso: $0 ARCHIVO.csv (debe existir y ser legible)" >&2
    exit 1
fi

creados=0
omitidos=0

while IFS=, read -r usuario grupo comentario; do
    [[ -z "$usuario" || "$usuario" == \#* ]] && continue

    if ! getent group "$grupo" > /dev/null; then
        groupadd "$grupo"
        echo "Grupo $grupo creado"
    fi

    if id "$usuario" &> /dev/null; then
        echo "Usuario $usuario ya existe, se omite"
        omitidos=$((omitidos + 1))
        continue
    fi

    useradd -m -G "$grupo" -c "$comentario" "$usuario"
    echo "$usuario:$CLAVE_INICIAL" | chpasswd
    chage -d 0 "$usuario"
    logger -t crear-usuarios "Usuario $usuario creado (grupo $grupo) por ${SUDO_USER:-root}"
    echo "Usuario $usuario creado en grupo $grupo"
    creados=$((creados + 1))
done < "$CSV"

echo "Resumen: $creados creado(s), $omitidos omitido(s)"
EOF
chmod +x ~/bin/crear-usuarios.sh
```

Qué observar: `useradd -G` agrega el grupo como suplementario (el usuario conserva su grupo privado, como en el Día 3); `chpasswd` recibe `usuario:clave` por stdin; `chage -d 0` pone la fecha del último cambio en 0 y fuerza el cambio al entrar; `$SUDO_USER` identifica quién ejecutó el `sudo` (se escribe `${SUDO_USER:-root}` porque con `set -u` la variable no existe si el script se ejecuta desde una sesión `sudo -i`).

3. Ejecutar (con ruta completa: `sudo` no usa el PATH del usuario) y verificar:

```bash
sudo ~/bin/crear-usuarios.sh ~/usuarios.csv
id mcastro
sudo chage -l lmartinez | head -2
sudo ~/bin/crear-usuarios.sh ~/usuarios.csv | tail -1
```

Salida esperada:
```
Grupo desarrollo creado
Usuario lmartinez creado en grupo desarrollo
Usuario rgomez creado en grupo desarrollo
Usuario jperez creado en grupo soporte
Usuario mcastro creado en grupo auditoria
Resumen: 4 creado(s), 0 omitido(s)
uid=2008(mcastro) gid=2008(mcastro) groups=2008(mcastro),3003(auditoria)
Last password change					: password must be changed
Password expires					: password must be changed
Resumen: 0 creado(s), 4 omitido(s)
```
Qué observar: los UID/GID concretos dependen de los que ya se usaron el Día 3 (`useradd` toma el siguiente UID libre; aquí 2005–2008 porque `laura` es 2004 y `temporal` se borró). El GID del grupo privado normalmente coincide con el UID, pero si en su VM sale distinto (por ejemplo 3005, porque `groupadd desarrollo` acaba de tomar el 3004) tampoco es un error: `useradd` elige el primer GID libre. ⚠️ Verificar en la VM antes de la clase y ajustar la salida esperada. Si `soporte` o `auditoria` no existían (Día 3 incompleto) también aparecen como "Grupo … creado". La segunda ejecución no rompe nada: el script es **idempotente**, palabra que reaparece con Ansible. Opcional: desde el equipo propio, `ssh -p 2222 lmartinez@localhost` con `Cambiar.2026` obliga a cambiar la contraseña de inmediato.

- **Checkpoint:** pegar en el chat la salida de:
```bash
getent group auditoria; sudo chage -l jperez | head -1
```

### Lab 3.2 — healthcheck.sh: estado del servidor con exit code (8 min)

- **Objetivo:** comprobar servicios (incluido `httpd` solo si está instalado), carga y memoria; imprimir OK/FALLA alineado con `printf` y devolver 0 si todo está bien o 1 si hay fallas.

1. El script (se ejecuta sin `sudo`):

```bash
cat > ~/bin/healthcheck.sh <<'EOF'
#!/bin/bash
# healthcheck.sh - revisión rápida: servicios, carga y memoria
# Uso:    healthcheck.sh
# Salida: 0 todo OK · 1 al menos una FALLA
SERVICIOS=(sshd chronyd crond)
rpm -q httpd &> /dev/null && SERVICIOS+=(httpd)
UMBRAL_MEM=90
fallas=0

estado() {
    printf "%-22s %-6s %s\n" "$1" "$2" "$3"
}

echo "=== Healthcheck de $(hostname) - $(date '+%F %T') ==="

for svc in "${SERVICIOS[@]}"; do
    if systemctl is-active --quiet "$svc"; then
        estado "Servicio $svc" "OK" "activo"
    else
        estado "Servicio $svc" "FALLA" "estado: $(systemctl is-active "$svc")"
        fallas=$((fallas + 1))
    fi
done

carga=$(awk '{print $1}' /proc/loadavg)
cpus=$(nproc)
if awk -v c="$carga" -v n="$cpus" 'BEGIN { exit (c > n) ? 0 : 1 }'; then
    estado "Carga (1 min)" "FALLA" "$carga con $cpus CPU"
    fallas=$((fallas + 1))
else
    estado "Carga (1 min)" "OK" "$carga con $cpus CPU"
fi

mem_pct=$(free -m | awk '/^Mem:/ {printf "%d", ($2 - $7) / $2 * 100}')
if [[ $mem_pct -ge $UMBRAL_MEM ]]; then
    estado "Memoria" "FALLA" "${mem_pct}% en uso (umbral ${UMBRAL_MEM}%)"
    fallas=$((fallas + 1))
else
    estado "Memoria" "OK" "${mem_pct}% en uso (umbral ${UMBRAL_MEM}%)"
fi

echo "Resultado: $fallas falla(s)"
[[ $fallas -eq 0 ]] && exit 0 || exit 1
EOF
chmod +x ~/bin/healthcheck.sh
healthcheck.sh; echo "código: $?"
```

Salida esperada:
```
=== Healthcheck de rhel01 - 2026-09-03 09:20:15 ===
Servicio sshd          OK     activo
Servicio chronyd       OK     activo
Servicio crond         OK     activo
Servicio httpd         OK     activo
Carga (1 min)          OK     0.05 con 2 CPU
Memoria                OK     21% en uso (umbral 90%)
Resultado: 0 falla(s)
código: 0
```
Qué observar: la línea de `httpd` solo aparece si el paquete está instalado (Día 4); si está instalado pero detenido, mostrará `FALLA` y el código será 1: es el comportamiento correcto. La carga se compara con `awk` porque Bash no maneja decimales; la memoria usa la columna `available` de `free` (columna 7), que es la medida real de memoria disponible.

2. Provocar una falla y comprobar el código de salida:

```bash
sudo systemctl stop chronyd
healthcheck.sh | grep -E "chronyd|Resultado"; echo "código: ${PIPESTATUS[0]}"
sudo systemctl start chronyd
```

Salida esperada:
```
Servicio chronyd       FALLA  estado: inactive
Resultado: 1 falla(s)
código: 1
```
Qué observar: `${PIPESTATUS[0]}` da el código del primer comando de la tubería (el de `grep` sería el de `$?`).

- **Checkpoint:** pegar en el chat la salida de `healthcheck.sh | tail -1`.

### Lab 3.3 — monitor-disco.sh: alerta de uso de disco (5 min)

- **Objetivo:** recorrer `df -h` con `awk`, avisar con `logger -p local0.warning` cuando un filesystem supere el umbral y escribir en `/var/log/monitor-disco.log`. Este script se instala en `/usr/local/bin` porque en el Bloque 4 lo ejecutará systemd como root.

1. El script:

```bash
cat > /tmp/monitor-disco.sh <<'EOF'
#!/bin/bash
# monitor-disco.sh - avisa cuando un filesystem supera el umbral de uso
# Uso:    monitor-disco.sh [UMBRAL]   (por defecto 80)
# Log:    /var/log/monitor-disco.log y syslog (local0.warning) vía logger
# Salida: 0 sin alertas · 1 con alertas
UMBRAL="${1:-80}"
LOG=/var/log/monitor-disco.log
FECHA=$(date '+%F %T')
alertas=0
revisados=0

while read -r punto uso; do
    revisados=$((revisados + 1))
    if [[ $uso -gt $UMBRAL ]]; then
        mensaje="ALERTA: $punto al ${uso}% (umbral ${UMBRAL}%)"
        logger -p local0.warning -t monitor-disco "$mensaje"
        alertas=$((alertas + 1))
    else
        mensaje="OK: $punto al ${uso}%"
    fi
    echo "$FECHA $mensaje" >> "$LOG"
done < <(df -h -x tmpfs -x devtmpfs -x efivarfs | awk 'NR > 1 {sub("%", "", $5); print $6, $5}')

echo "Revisados $revisados filesystems, $alertas alerta(s)"
[[ $alertas -eq 0 ]] && exit 0 || exit 1
EOF
sudo cp /tmp/monitor-disco.sh /usr/local/bin/monitor-disco.sh
sudo chmod 755 /usr/local/bin/monitor-disco.sh
ls -lZ /usr/local/bin/monitor-disco.sh
```

Salida esperada:
```
-rwxr-xr-x. 1 root root unconfined_u:object_r:bin_t:s0 897 Sep  3 09:28 /usr/local/bin/monitor-disco.sh
```
Qué observar: en `ls -lZ` el contexto SELinux se imprime **entre el grupo y el tamaño** (no al final). Se copia con `cp` (no `mv`) para que el archivo reciba el contexto SELinux `bin_t` del directorio destino; un `mv` desde `/home` arrastraría `user_home_t` y systemd podría negarse a ejecutarlo. `awk` quita el `%` con `sub()` y deja dos columnas: punto de montaje y porcentaje. El `while` lee de `< <(…)` para que `alertas` sobreviva al bucle.

2. Ejecutar con el umbral normal y con uno absurdo para forzar alertas:

```bash
sudo /usr/local/bin/monitor-disco.sh; echo "código: $?"
sudo /usr/local/bin/monitor-disco.sh 5; echo "código: $?"
sudo tail -4 /var/log/monitor-disco.log
sudo journalctl -t monitor-disco -p warning -n 2 --no-pager
```

Salida esperada:
```
Revisados 3 filesystems, 0 alerta(s)
código: 0
Revisados 3 filesystems, 2 alerta(s)
código: 1
2026-09-03 09:30:02 OK: /datos al 1%
2026-09-03 09:30:40 ALERTA: / al 18% (umbral 5%)
2026-09-03 09:30:40 ALERTA: /boot al 25% (umbral 5%)
2026-09-03 09:30:40 OK: /datos al 1%
Sep 03 09:30:40 rhel01 monitor-disco[5820]: ALERTA: / al 18% (umbral 5%)
Sep 03 09:30:40 rhel01 monitor-disco[5822]: ALERTA: /boot al 25% (umbral 5%)
```
Qué observar: con umbral 5 alertan `/` y `/boot`, pero no `/datos` (1 no es mayor que 5). El número de "Revisados" depende de lo que quedó montado el Día 6: si además están `/backups` y `/stratis`, serán 5; si `/datos` es un directorio dentro de `/` (Día 6 incompleto), no aparece como línea aparte. En UTM (arranque UEFI) aparece también `/boot/efi`. En VirtualBox los dispositivos son `/dev/sda*`, en UTM `/dev/vda*`; el script no depende de eso porque solo mira punto de montaje y porcentaje.

- **Checkpoint:** pegar en el chat la salida de `sudo tail -1 /var/log/monitor-disco.log`.

---

## Bloque 4 — Programación de tareas: cron, at y systemd timers

### Conceptos (10 min)

**Tres herramientas, tres preguntas.** ¿La tarea se repite? → **cron** (o un **systemd timer**). ¿Se ejecuta una sola vez, más tarde? → **at**. ¿Es una tarea del sistema que debe integrarse con systemd (dependencias, logs en el journal, recuperar ejecuciones perdidas)? → **systemd timer**. Red Hat migra sus propias tareas a timers (`dnf-makecache.timer`, `logrotate.timer`, `systemd-tmpfiles-clean.timer`), pero cron sigue siendo el estándar para tareas de usuarios y aplicaciones, y es lo que pide el RHCSA.

| | cron | at / batch | systemd timer |
|---|---|---|---|
| Uso | recurrente | una vez (`at`) o cuando baje la carga (`batch`) | recurrente o relativo al arranque |
| Quién lo define | cada usuario (`crontab -e`) o root (`/etc/cron.d/`) | cada usuario | root en `/etc/systemd/system/` |
| Si el equipo estaba apagado | se pierde (anacron cubre daily/weekly/monthly del sistema) | los trabajos vencidos se ejecutan al iniciar `atd` | `Persistent=true` lo ejecuta al arrancar |
| Logs | `/var/log/cron`, `journalctl -u crond` | correo local | `journalctl -u nombre.service` |

**cron.** El demonio `crond` (paquete `cronie`) lee cada minuto los crontabs. Formato de una línea: cinco campos de tiempo y el comando:

```text
#  campo 1  campo 2  campo 3     campo 4  campo 5     comando
#  minuto   hora     día del mes mes      día semana
#  0-59     0-23     1-31        1-12     0-7 (0 y 7 = domingo)
   *        *        *           *        *           comando

   */5      *        *           *        *      cada 5 minutos
   30       23       *           *        *      todos los días a las 23:30
   0        2        *           *        1-5    lunes a viernes a las 02:00
   0        8,13     *           *        *      a las 08:00 y a las 13:00
   15       3        1           *        *      el día 1 de cada mes a las 03:15
   @reboot                                       al arrancar (también @hourly @daily @weekly @monthly @yearly)
```
Los operadores de los campos son siempre los mismos: `*` (todos), `,` (lista: `8,13`), `-` (rango: `1-5`) y `/` (paso: `*/5`, `0/10`). Si se rellenan a la vez **día del mes** y **día de la semana**, cron ejecuta cuando se cumple **cualquiera** de los dos, no los dos a la vez: es la trampa clásica de `0 3 1 * 1` ("el día 1 **o** los lunes").

- `crontab -e` edita, `-l` lista, `-r` **borra sin preguntar** (está al lado de la `e`: hacer `crontab -l > ~/crontab.bak` antes). `sudo crontab -u ana -l` administra el de otro usuario. Los crontabs se guardan en `/var/spool/cron/USUARIO` (no editar a mano).
- Variables al inicio del crontab: `SHELL=/bin/bash`, `PATH=…`, `MAILTO=""`. **La trampa número uno**: cron ejecuta con un PATH mínimo (`/usr/bin:/bin`) y sin el entorno del usuario; un script que funciona a mano falla en cron con `command not found`. Solución: rutas absolutas o `PATH` en el crontab.
- La salida (stdout/stderr) se envía por correo al usuario; en RHEL 9 no hay servidor de correo instalado, así que **se pierde**. Redirigir siempre: `>> /ruta/log 2>&1`.
- Cron del sistema: `/etc/crontab` y `/etc/cron.d/*` tienen un **sexto campo con el usuario** que ejecuta. `/etc/cron.hourly/`, `cron.daily/`, `cron.weekly/`, `cron.monthly/` contienen scripts (no crontabs) que ejecuta `run-parts`: deben tener permiso de ejecución y **no llevar punto en el nombre** (`run-parts` ignora `respaldo.sh`; hay que llamarlo `respaldo`). Del mismo modo, `crond` ignora los archivos de `/etc/cron.d/` cuyo nombre contenga un punto. El horario lo controla `/etc/cron.d/0hourly` (cada hora) y **anacron** (`/etc/anacrontab`) para daily/weekly/monthly: anacron recuerda cuándo fue la última ejecución y, si el servidor estuvo apagado, la ejecuta al encender (con un retraso aleatorio). Ideal para servidores que no están encendidos 24/7; en un servidor siempre encendido, cron y anacron se comportan igual.
- Control de acceso: si existe `/etc/cron.allow`, solo esos usuarios pueden usar `crontab`; si no, se niega a los listados en `/etc/cron.deny` (existe vacío por defecto). root siempre puede.
- Logs: `/var/log/cron` (vía rsyslog) y `journalctl -u crond`. Cada ejecución aparece como `(usuario) CMD (comando)`.

**at.** `atd` ejecuta trabajos únicos. `at 17:30`, `at now + 5 minutes`, `at 02:00 tomorrow`, `at 10:00 2026-12-24`; se escriben los comandos y se cierra con Ctrl+D. `atq` lista, `atrm N` elimina, `at -c N` muestra el trabajo completo (incluido el entorno capturado). `batch` ejecuta cuando la carga del sistema es baja. `/etc/at.allow` y `/etc/at.deny` funcionan igual que en cron. En RHEL 9 el paquete `at` puede no estar instalado ni `atd` activo: verificar.

**systemd timers.** Dos unidades con el mismo nombre: `nombre.service` (qué se ejecuta, normalmente `Type=oneshot`) y `nombre.timer` (cuándo). Se habilita y arranca **el timer**, no el servicio. `OnCalendar=` usa la sintaxis de `systemd.time(7)`: `DíaSemana Año-Mes-Día Hora:Min:Seg`, y se comprueba con `systemd-analyze calendar`.

| `OnCalendar=` | Significado |
|---|---|
| `*:0/10` | cada 10 minutos |
| `hourly` · `daily` · `weekly` | cada hora en punto · cada día a las 00:00 · lunes 00:00 |
| `*-*-* 02:00` | todos los días a las 02:00 |
| `Mon..Fri 02:00` | lunes a viernes a las 02:00 |
| `Sat *-*-* 23:00` | sábados a las 23:00 |
| `*-*-01 03:00` | el día 1 de cada mes a las 03:00 |

También existen timers relativos (`OnBootSec=5min`, `OnUnitActiveSec=1h`) para "5 minutos tras arrancar y luego cada hora". `Persistent=true` guarda la última ejecución en disco y, si se perdió una por apagado, la lanza al arrancar. `systemctl list-timers` muestra próxima y última ejecución; los logs van al journal del servicio.

**systemd-tmpfiles.** Crea, ajusta y limpia archivos y directorios según reglas en `/usr/lib/tmpfiles.d/` (del sistema, no editar) y `/etc/tmpfiles.d/` (del administrador). Se ejecuta al arrancar (`systemd-tmpfiles-setup.service`) y a diario (`systemd-tmpfiles-clean.timer`). Así se limpian `/tmp` (10 días) y `/var/tmp` (30 días) en RHEL 9. Formato: `Tipo Ruta Modo Usuario Grupo Edad`. `d` crea el directorio si no existe y limpia lo que sea más antiguo que la edad; `q` hace lo mismo pero creando un subvolumen en btrfs (en XFS, que es lo que usa esta VM, se comporta exactamente como `d`); `L` crea un enlace simbólico; `z` ajusta modo, dueño y contexto SELinux de algo que ya existe (`Z` lo hace de forma recursiva); `x`/`X` excluyen rutas de la limpieza.

### Lab 4.1 — cron y anacron (15 min)

- **Objetivo:** programar tareas de usuario, caer en la trampa del PATH y salir de ella, revisar los logs, conocer los archivos del sistema y el control de acceso.

1. Estado del servicio y archivos del sistema:

```bash
systemctl is-enabled crond; systemctl is-active crond
rpm -q cronie cronie-anacron
ls -d /etc/cron*
```

Salida esperada:
```
enabled
active
cronie-1.5.7-11.el9.x86_64
cronie-anacron-1.5.7-11.el9.x86_64
/etc/cron.d  /etc/cron.daily  /etc/cron.deny  /etc/cron.hourly  /etc/cron.monthly  /etc/crontab  /etc/cron.weekly
```
(La versión exacta y la arquitectura varían; en UTM termina en `aarch64`.)

2. Un crontab de prueba con la trampa del PATH incluida (se instala desde un archivo con `crontab ARCHIVO` para que todos tengan lo mismo; `crontab -e` abre el editor de `$EDITOR`, que en RHEL 9 es `nano` si el paquete `nano` está instalado —lo trae la instalación tipo Server y define `EDITOR=/usr/bin/nano` en `/etc/profile.d/nano-default-editor.sh`— y `vi` si no lo está; quien prefiera `vim` ejecuta antes `export EDITOR=vim`):

```bash
crontab -l
cat > ~/crontab-prueba <<'EOF'
* * * * * date >> /home/student/cron-prueba.txt
* * * * * healthcheck.sh >> /home/student/cron-hc.txt 2>&1
EOF
crontab ~/crontab-prueba
crontab -l
```

Salida esperada:
```
no crontab for student
* * * * * date >> /home/student/cron-prueba.txt
* * * * * healthcheck.sh >> /home/student/cron-hc.txt 2>&1
```

3. Mientras pasa el minuto, revisar el log del sistema; después, los resultados:

```bash
sleep 65; cat ~/cron-prueba.txt; cat ~/cron-hc.txt
sudo tail -3 /var/log/cron
```

Salida esperada:
```
Thu Sep  3 09:46:01 AM EST 2026
/bin/sh: line 1: healthcheck.sh: command not found
Sep  3 09:46:01 rhel01 CROND[6210]: (student) CMD (date >> /home/student/cron-prueba.txt)
Sep  3 09:46:01 rhel01 CROND[6211]: (student) CMD (healthcheck.sh >> /home/student/cron-hc.txt 2>&1)
Sep  3 09:46:01 rhel01 CROND[6205]: (student) CMDEND (date >> /home/student/cron-prueba.txt)
```
Qué observar: en RHEL 9 la salida de `date` sin formato incluye `AM`/`PM` (cambio de glibc 2.34 respecto a RHEL 8). Las tres líneas de `/var/log/cron` son un ejemplo: según el momento exacto se verán `CMD` y `CMDEND` de la última o de las dos últimas ejecuciones, y si el minuto ya pasó dos veces habrá más. Lo importante: cron **sí** ejecutó `healthcheck.sh` (aparece en el log) pero la shell de cron (`/bin/sh`, PATH mínimo) no lo encontró. A mano funciona porque `~/bin` está en el PATH interactivo. El mismo script, dos resultados: la causa es el entorno, no el script.

4. El crontab definitivo del usuario, con variables y rutas absolutas:

```bash
cat > ~/mi-crontab <<'EOF'
SHELL=/bin/bash
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""
# m  h  dom mon dow  comando
# Respaldo diario de ~/empresa a las 23:30
30 23 *   *   *    /home/student/bin/backup.sh /home/student/empresa >> /home/student/backup.log 2>&1
# Healthcheck cada 15 min en horario laboral, lunes a viernes
*/15 8-17 * *  1-5  healthcheck.sh >> /home/student/healthcheck.log 2>&1
# Aviso al arrancar
@reboot            logger -t cron-student "El servidor arrancó y crond ya ejecuta tareas de student"
EOF
crontab ~/mi-crontab
crontab -l | grep -v '^#'
rm -f ~/cron-prueba.txt ~/cron-hc.txt ~/crontab-prueba
```

Salida esperada:
```
SHELL=/bin/bash
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""
30 23 *   *   *    /home/student/bin/backup.sh /home/student/empresa >> /home/student/backup.log 2>&1
*/15 8-17 * *  1-5  healthcheck.sh >> /home/student/healthcheck.log 2>&1
@reboot            logger -t cron-student "El servidor arrancó y crond ya ejecuta tareas de student"
```
Qué observar: `crontab ARCHIVO` **reemplaza** todo el crontab (como haría `crontab -r` seguido de una nueva edición). `MAILTO=""` evita intentos de correo. Con `PATH` definido, `healthcheck.sh` ya se encuentra; aun así, la costumbre profesional es la ruta absoluta.

5. Cron del sistema: leer los archivos y crear una tarea en `/etc/cron.d/`:

```bash
cat /etc/crontab
cat /etc/cron.d/0hourly
ls /etc/cron.hourly/ /etc/cron.daily/
grep -v '^#' /etc/anacrontab
sudo cp ~/bin/healthcheck.sh /usr/local/bin/healthcheck.sh
sudo tee /etc/cron.d/healthcheck > /dev/null <<'EOF'
# Healthcheck del servidor cada hora; formato de sistema: incluye el usuario
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 * * * * root /usr/local/bin/healthcheck.sh >> /var/log/healthcheck.log 2>&1
EOF
ls -l /etc/cron.d/healthcheck
```

Salida esperada (resumida):
```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# For details see man 4 crontabs
# Example of job definition:
# .---------------- minute (0 - 59)
# ...
# *  *  *  *  * user-name  command to be executed
# Run the hourly jobs
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
01 * * * * root run-parts /etc/cron.hourly
/etc/cron.hourly/:
0anacron
/etc/cron.daily/:
SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
RANDOM_DELAY=45
START_HOURS_RANGE=3-22
1	5	cron.daily		nice run-parts /etc/cron.daily
7	25	cron.weekly		nice run-parts /etc/cron.weekly
@monthly 45	cron.monthly		nice run-parts /etc/cron.monthly
-rw-r--r--. 1 root root 221 Sep  3 09:52 /etc/cron.d/healthcheck
```
Qué observar: `/etc/crontab` en RHEL está vacío de tareas: solo define variables y documenta el formato. `0hourly` lanza `run-parts /etc/cron.hourly` al minuto 01, y ahí `0anacron` decide si toca ejecutar daily/weekly/monthly (con retraso aleatorio de hasta 45 min y solo entre las 3 y las 22 h). `/etc/cron.daily/` puede estar vacío: en RHEL 9 `logrotate` ya usa un timer de systemd. Los archivos de `/etc/cron.d/` no necesitan recargar `crond`: los detecta solo.

6. Control de acceso con `cron.deny` (usando `jperez`, creado en el Lab 3.1). Antes se "normaliza" la contraseña de `jperez`: `crontab` pasa por PAM (`/etc/pam.d/crond`) y una cuenta con cambio de contraseña pendiente (`chage -d 0`) puede ser rechazada con un mensaje distinto al que queremos ver:

```bash
sudo chage -d "$(date +%F)" jperez
cat /etc/cron.deny; ls /etc/cron.allow
echo jperez | sudo tee -a /etc/cron.deny
sudo -u jperez crontab -l
sudo sed -i '/^jperez$/d' /etc/cron.deny
sudo -u jperez crontab -l
```

Salida esperada:
```
ls: cannot access '/etc/cron.allow': No such file or directory
jperez
You (jperez) are not allowed to use this program (crontab)
See crontab(1) for more information
no crontab for jperez
```
Qué observar: `sudo -u jperez` ejecuta el comando como ese usuario sin pedir su contraseña (la de `student` ya está en caché). Con `jperez` en `cron.deny` el rechazo es inmediato; al quitarlo, `crontab -l` responde con normalidad. ⚠️ Verificar en la VM antes de la clase: si aun así aparece `You (jperez) are not allowed to access to (crontab) because of pam configuration`, revisar `sudo passwd -S jperez` y `sudo chage -l jperez`.

- **Checkpoint:** pegar en el chat la salida de:
```bash
crontab -l | grep -c -v '^#'; sudo grep -c "(student) CMD" /var/log/cron
```

### Lab 4.2 — at y batch (8 min)

- **Objetivo:** instalar y activar `atd`, programar un trabajo único, listarlo, inspeccionarlo y eliminarlo.

1. Verificar/instalar y activar:

```bash
rpm -q at || sudo dnf install -y at
sudo systemctl enable --now atd
systemctl is-active atd
```

Salida esperada (si `at` ya estaba instalado, la primera línea muestra la versión, por ejemplo `at-3.1.23-11.el9.x86_64`; si no, `dnf` imprime la transacción de instalación):
```
Created symlink /etc/systemd/system/multi-user.target.wants/atd.service → /usr/lib/systemd/system/atd.service.
active
```
Qué observar: el paquete `at` trae un *preset* que habilita `atd`, así que en algunas instalaciones el servicio ya está habilitado y `enable --now` no imprime la línea `Created symlink…`; lo que importa es que `systemctl is-active atd` responda `active`.

2. Programar un trabajo para dentro de 2 minutos (los comandos se escriben en el prompt `at>` y se termina con **Ctrl+D**):

```bash
at now + 2 minutes
```
```text
warning: commands will be executed using /bin/sh
at> echo "Trabajo at ejecutado: $(date)" >> /home/student/at-prueba.txt
at> logger -t at-demo "trabajo de at ejecutado por $USER"
at> <Ctrl+D>
job 1 at Thu Sep  3 10:02:00 2026
```

3. Otras formas de programar, listar, inspeccionar y eliminar:

```bash
echo "logger -t at-demo 'recordatorio de las 17:30'" | at 17:30
atq
at -c 2 | tail -8
atrm 2
atq
```

Salida esperada (resumida; `at -c` imprime primero decenas de líneas con las variables de entorno, y el delimitador `marcinDELIMITER…` varía):
```
warning: commands will be executed using /bin/sh
job 2 at Thu Sep  3 17:30:00 2026
1	Thu Sep  3 10:02:00 2026 a student
2	Thu Sep  3 17:30:00 2026 a student
cd /home/student || {
	 echo 'Execution directory inaccessible' >&2
	 exit 1
}
${SHELL:-/bin/sh} << 'marcinDELIMITER5f3a9c1e'
logger -t at-demo 'recordatorio de las 17:30'

marcinDELIMITER5f3a9c1e
1	Thu Sep  3 10:02:00 2026 a student
```
Qué observar: `at -c` muestra que el trabajo guarda el entorno (variables como `PATH`, directorio de trabajo) del momento en que se creó; por eso `at` no sufre la trampa del PATH tanto como cron. Si la clase es después de las 17:30, el trabajo 2 queda para el día siguiente: la salida lo indica. ⚠️ Verificar en la VM antes de la clase el número exacto de líneas finales que muestra `at -c` en la versión instalada de `at`. `batch` se usa igual (`echo "comando" | batch`) y se ejecuta cuando baja la carga. Cuando pasen los 2 minutos: `cat ~/at-prueba.txt` y `sudo journalctl -t at-demo -n 1`.

- **Checkpoint:** pegar en el chat la salida de `systemctl is-active atd; atq`.

### Lab 4.3 — systemd timer para monitor-disco (12 min)

- **Objetivo:** crear `monitor-disco.service` (oneshot) y `monitor-disco.timer` (cada 10 minutos, persistente), activarlo, comprobar `list-timers`, ejecutar a mano y leer el journal.

1. Ver los timers que RHEL 9 ya trae:

```bash
systemctl list-timers --no-pager | head -6
```

Salida esperada (resumida):
```
NEXT                        LEFT       LAST                        PASSED  UNIT                         ACTIVATES
Thu 2026-09-03 10:12:33 EST 7min left  Thu 2026-09-03 09:12:33 EST 52min ago dnf-makecache.timer        dnf-makecache.service
Thu 2026-09-03 10:20:10 EST 15min left Thu 2026-09-03 09:20:10 EST 45min ago systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Fri 2026-09-04 00:00:00 EST 13h left   Thu 2026-09-03 08:01:12 EST 2h ago    logrotate.timer            logrotate.service
Mon 2026-09-07 00:52:11 EST 3 days left Mon 2026-08-31 00:30:00 EST 3 days ago fstrim.timer              fstrim.service
```

2. Crear el servicio y el timer:

```bash
sudo tee /etc/systemd/system/monitor-disco.service > /dev/null <<'EOF'
[Unit]
Description=Monitor de uso de disco (PGN)
Documentation=man:df(1)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-disco.sh 80
EOF

sudo tee /etc/systemd/system/monitor-disco.timer > /dev/null <<'EOF'
[Unit]
Description=Ejecuta monitor-disco cada 10 minutos

[Timer]
OnCalendar=*:0/10
Persistent=true
Unit=monitor-disco.service

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now monitor-disco.timer
systemctl list-timers monitor-disco.timer --no-pager
```

Salida esperada:
```
Created symlink /etc/systemd/system/timers.target.wants/monitor-disco.timer → /etc/systemd/system/monitor-disco.timer.
NEXT                        LEFT      LAST PASSED UNIT                ACTIVATES
Thu 2026-09-03 10:10:00 EST 4min left n/a  n/a    monitor-disco.timer monitor-disco.service

1 timers listed.
```
Qué observar: el servicio **no** tiene sección `[Install]` ni se habilita: lo dispara el timer. `Type=oneshot` indica que el proceso termina por sí mismo. Si el servicio no aparece o se editó después, repetir `daemon-reload`.

3. Ejecutar el servicio a mano (sin esperar los 10 minutos) y leer el journal:

```bash
sudo systemctl start monitor-disco.service
sudo journalctl -u monitor-disco.service --no-pager -n 4
systemctl status monitor-disco.timer --no-pager | head -5
```

Salida esperada:
```
Sep 03 10:06:12 rhel01 systemd[1]: Starting Monitor de uso de disco (PGN)...
Sep 03 10:06:12 rhel01 monitor-disco.sh[6810]: Revisados 3 filesystems, 0 alerta(s)
Sep 03 10:06:12 rhel01 systemd[1]: monitor-disco.service: Deactivated successfully.
Sep 03 10:06:12 rhel01 systemd[1]: Finished Monitor de uso de disco (PGN).
● monitor-disco.timer - Ejecuta monitor-disco cada 10 minutos
     Loaded: loaded (/etc/systemd/system/monitor-disco.timer; enabled; preset: disabled)
     Active: active (waiting) since Thu 2026-09-03 10:05:40 EST; 1min ago
    Trigger: Thu 2026-09-03 10:10:00 EST; 3min left
   Triggers: ● monitor-disco.service
```
Qué observar: la salida estándar del script va al journal automáticamente (no hace falta redirigir como en cron). Cuando el script alerte, las líneas de `logger -p local0.warning` también aparecen en `journalctl -u monitor-disco.service`.

4. Comprobar expresiones de calendario antes de usarlas:

```bash
systemd-analyze calendar "Mon..Fri 08:00"
systemd-analyze calendar --iterations=3 "*:0/10" | grep -E "Normalized|Next|Iter"
```

Salida esperada:
```
  Original form: Mon..Fri 08:00
Normalized form: Mon..Fri *-*-* 08:00:00
    Next elapse: Fri 2026-09-04 08:00:00 EST
       (in UTC): Fri 2026-09-04 13:00:00 UTC
       From now: 21h left
Normalized form: *-*-* *:00/10:00
    Next elapse: Thu 2026-09-03 10:10:00 EST
       Iter. #2: Thu 2026-09-03 10:20:00 EST
       Iter. #3: Thu 2026-09-03 10:30:00 EST
```

- **Checkpoint:** pegar en el chat la salida de:
```bash
systemctl list-timers monitor-disco.timer --no-pager | head -2
```

### Lab 4.4 — systemd-tmpfiles: limpieza de /datos/tmp (5 min)

- **Objetivo:** definir una regla que garantice la existencia de `/datos/tmp` con permisos 1777 y borre lo que tenga más de un día; ver cómo lo hace RHEL con `/tmp`.

1. Reglas del sistema y regla propia:

```bash
grep -v '^#' /usr/lib/tmpfiles.d/tmp.conf
sudo tee /etc/tmpfiles.d/datos-tmp.conf > /dev/null <<'EOF'
# Tipo Ruta        Modo Usuario Grupo Edad
d     /datos/tmp   1777 root    root  1d
EOF
sudo systemd-tmpfiles --create /etc/tmpfiles.d/datos-tmp.conf
ls -ld /datos/tmp
```

Salida esperada (además de algunas líneas en blanco):
```
q /tmp 1777 root root 10d
q /var/tmp 1777 root root 30d
x /tmp/systemd-private-%b-*
X /tmp/systemd-private-%b-*/tmp
x /var/tmp/systemd-private-%b-*
X /var/tmp/systemd-private-%b-*/tmp
drwxrwxrwt. 2 root root 6 Sep  3 08:05 /datos/tmp
```
Qué observar: las líneas `x`/`X` excluyen de la limpieza los directorios privados que systemd crea para los servicios con `PrivateTmp=yes` (no tocar). El directorio `/datos/tmp` existía con 755 (lo creamos al inicio del día) y `--create` lo corrigió a 1777 (bit sticky, como `/tmp`).

2. Probar la limpieza y ver quién la ejecuta a diario:

```bash
touch /datos/tmp/reciente.txt
sudo systemd-tmpfiles --clean /etc/tmpfiles.d/datos-tmp.conf
ls /datos/tmp
systemctl cat systemd-tmpfiles-clean.timer | grep -E "OnBootSec|OnUnitActiveSec"
```

Salida esperada:
```
reciente.txt
OnBootSec=15min
OnUnitActiveSec=1d
```
Qué observar: `reciente.txt` sobrevive porque tiene menos de un día. Para que un archivo se borre, sus tres marcas de tiempo (mtime, atime y ctime) deben ser más antiguas que la edad indicada; `touch -d "-2 days"` cambia mtime y atime pero **no** ctime, por lo que no sirve para simular la limpieza en clase: en producción los archivos envejecen solos y el timer diario los borra. `systemd-tmpfiles --clean` sin argumentos aplica todas las reglas del sistema.

- **Checkpoint:** pegar en el chat la salida de `ls -ld /datos/tmp; cat /etc/tmpfiles.d/datos-tmp.conf`.

---

## Bloque 5 — Automatización empresarial

Este bloque es contexto profesional (no está en el EX200) y es el primero que se recorta si el Bloque 4 se alarga: en ese caso, los Conceptos se dan en 5 minutos y el Lab 5.1 y la demo de Cockpit quedan como tarea con la salida esperada publicada en el chat.

### Conceptos (5 min)

**Qué decir en clase.** Todo lo de hoy automatiza **un** servidor. Cuando son 30 o 300, escribir un script y copiarlo a mano deja de escalar: hay que describir **el estado deseado** de los servidores en archivos de texto, guardarlos en un repositorio (Git) y dejar que una herramienta los aplique. Eso es **infraestructura como código**: la configuración se revisa, se versiona y se repite igual en todos los equipos.

- **Ansible** es la herramienta de Red Hat para esto. No necesita agente: se conecta por SSH y usa el Python del servidor destino. Vocabulario mínimo: **inventario** (lista de servidores), **módulo** (una acción: `dnf`, `copy`, `service`, `user`…), **tarea** (un módulo con parámetros), **playbook** (archivo YAML con tareas ordenadas). La propiedad clave es la **idempotencia**: ejecutar el playbook dos veces deja el mismo resultado; la segunda vez no cambia nada (`changed=0`). Compárelo con `crear-usuarios.sh`, que también era idempotente… porque lo programamos así; en Ansible viene de fábrica.
- `ansible-core` (el motor, gratuito) está en el repositorio AppStream de RHEL 9. **Red Hat Ansible Automation Platform** es el producto empresarial: interfaz web, control de acceso, ejecución programada, inventarios dinámicos, colecciones certificadas. El curso completo es **RH294** (y su examen, EX294, es el RHCE).
- Otras herramientas de administración a escala: **Red Hat Satellite** (repositorios y parches centralizados, aprovisionamiento, cumplimiento; los servidores se registran contra Satellite en lugar de contra Red Hat), **Red Hat Insights** (servicio en la nube incluido en la suscripción que analiza la configuración y avisa de riesgos: `insights-client --register`), y **Cockpit** (consola web incluida en RHEL, puerto 9090: estado del sistema, logs, almacenamiento, red, usuarios, servicios, terminal; ideal para técnicos que aún no dominan la terminal y para diagnósticos rápidos).

### Lab 5.1 — Ansible: inventario y primer playbook (7 min)

- **Objetivo:** instalar `ansible-core`, apuntar el inventario a la propia VM, ejecutar un playbook que instala `tree` y crea un archivo, y observar la idempotencia. Para ganar tiempo, el instructor pega en el chat los tres bloques completos; los participantes los ejecutan mientras se explican.

1. Instalar y crear el inventario:

```bash
sudo dnf install -y ansible-core
ansible --version | head -1
mkdir -p ~/ansible && cd ~/ansible
cat > inventario <<'EOF'
[servidores]
localhost ansible_connection=local
EOF
ansible -i inventario all -m ping
```

Salida esperada:
```
ansible [core 2.14.17]
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
Qué observar: la versión de `ansible-core` depende de la 9.x instalada (2.14.x en RHEL 9.2–9.4 y 2.16.x a partir de 9.5; ⚠️ Verificar en la VM antes de la clase con `ansible --version` y ajustar la salida esperada). `ansible_connection=local` evita SSH para este ejercicio. Con más servidores, cada línea sería un hostname o IP y Ansible entraría por SSH con la llave del Día 5.

2. El playbook (YAML: la indentación con espacios es parte de la sintaxis):

```bash
cat > base.yml <<'EOF'
---
- name: Configuración base de un servidor de la PGN
  hosts: servidores
  become: true
  tasks:
    - name: Asegurar que tree está instalado
      ansible.builtin.dnf:
        name: tree
        state: present

    - name: Crear mensaje del día que identifica la gestión con Ansible
      ansible.builtin.copy:
        dest: /etc/motd
        content: "Servidor {{ ansible_hostname }} gestionado con Ansible - PGN Informatica\n"
EOF
ansible-playbook -i inventario base.yml -K
```

Salida esperada (pide la contraseña de `sudo` en `BECOME password:`):
```
PLAY [Configuración base de un servidor de la PGN] *****************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Asegurar que tree está instalado] ****************************************
ok: [localhost]

TASK [Crear mensaje del día que identifica la gestión con Ansible] *************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost   : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Qué observar: `tree` ya estaba instalado desde el Día 2, por eso `ok` y no `changed` (si alguien lo desinstaló, verá `changed` y `changed=2`); el `motd` sí cambió. `-K` pide la contraseña de `sudo` porque `become: true` eleva privilegios.

3. Idempotencia: ejecutar de nuevo y comprobar el archivo:

```bash
ansible-playbook -i inventario base.yml -K | tail -2
cat /etc/motd
```

Salida esperada:
```
localhost   : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
Servidor rhel01 gestionado con Ansible - PGN Informatica
```
Qué observar: `changed=0`. El mismo playbook, aplicado a 300 servidores, produce 300 servidores iguales. En la próxima sesión SSH aparecerá el mensaje del `motd`.

- **Checkpoint:** pegar en el chat la línea `PLAY RECAP` de la segunda ejecución.

### Demo del instructor — Cockpit (3 min)

Hay **dos formas** de llegar a la consola desde el equipo propio; la segunda no toca la configuración del hipervisor y es la recomendada porque se practicó el Día 5:

1. Regla de port forwarding adicional en la VM: **9090 → 9090** (VirtualBox: Configuración → Red → Adaptador 1 → Avanzadas → Reenvío de puertos; UTM: configuración del dispositivo de red en modo *Emulated VLAN* → Reenvío de puertos). Requiere apagar la VM en VirtualBox si el adaptador no admite el cambio en caliente.
2. **Túnel SSH** (Día 5, sin tocar la VM ni el hipervisor), desde una terminal del equipo propio: `ssh -p 2222 -L 9090:localhost:9090 -N student@localhost` y dejarla abierta.

```bash
rpm -q cockpit || sudo dnf install -y cockpit
sudo systemctl enable --now cockpit.socket
systemctl is-active cockpit.socket
sudo firewall-cmd --list-services
```

Salida esperada:
```
cockpit-311.1-1.el9.x86_64
active
cockpit dhcpv6-client ssh
```
Qué observar: la versión exacta de `cockpit` varía con la 9.x instalada. En la instalación tipo **Server** de RHEL 9, `cockpit` viene instalado y `cockpit.socket` suele estar **ya habilitado** (se vio el Día 4 como `active listening`): en ese caso `enable --now` no imprime nada y no aparece la línea `Created symlink …`. Solo si estuviera deshabilitado se vería `Created symlink /etc/systemd/system/sockets.target.wants/cockpit.socket → /usr/lib/systemd/system/cockpit.socket.`

En el navegador del equipo propio: `https://localhost:9090` → aceptar el certificado autofirmado → usuario `student` → activar "Acceso administrativo" (botón "Limited access"/"Turn on administrative access"). Mostrar: **Overview** (carga, memoria, salud), **Logs** (el journal filtrable: buscar `monitor-disco`), **Services** (buscar `monitor-disco.timer`: pestaña Timers), **Storage** (los discos del Día 6), **Accounts** (los usuarios del Lab 3.1 con "contraseña debe cambiarse"), **Terminal**. Mensaje: Cockpit no reemplaza la terminal; la complementa y es un buen apoyo para el equipo de soporte. El firewall ya permite `cockpit` por defecto en la zona `public`, así que no hay nada que abrir. Si en alguna VM el socket estuviera apagado, el mensaje "Activate the web console with: systemctl enable --now cockpit.socket" aparece al iniciar sesión por SSH.

---

## Reto individual (25 min)

**Ticket PGN-1187 — Limpieza de logs de la aplicación de expedientes**

> "El servidor de expedientes se llena de archivos `.log` en `/home/student/empresa/logs/app`. Necesitamos un script `limpiar-logs.sh` que reciba **dos argumentos**: el directorio y una cantidad de días. Debe validar que el directorio existe y que los días son un número (si no, mensaje de error y salir con código 1). Tiene que comprimir con `gzip` los `.log` con más de N días de antigüedad y borrar los `.gz` con más de 30 días. Debe registrar con `logger` cuántos archivos comprimió y cuántos borró. Hay que programarlo a las 02:00 de lunes a viernes de dos formas: en cron **y** como systemd timer equivalente. Demuestre que funciona con datos de prueba."

Datos de prueba (todos ejecutan lo mismo):

```bash
mkdir -p ~/empresa/logs/app && cd ~/empresa/logs/app
for i in {1..5}; do echo "log reciente $i" > reciente-$i.log; done
for i in {1..5}; do echo "log viejo $i" > viejo-$i.log; touch -d "-10 days" viejo-$i.log; done
for i in {1..3}; do echo "gz antiguo $i" | gzip > antiguo-$i.log.gz; touch -d "-45 days" antiguo-$i.log.gz; done
ls -l --time-style=+%F | awk 'NR>1 {print $6}' | sort | uniq -c
cd ~
```

Salida esperada (las fechas dependen del día: hace 45 días, hace 10 días y hoy):
```
      3 2026-07-20
      5 2026-08-24
      5 2026-09-03
```

Entregable: el script, la salida de `ls ~/empresa/logs/app` después de ejecutarlo, la línea del crontab (`crontab -l`), `systemctl list-timers limpiar-logs.timer` y la entrada del journal generada por `logger`.

### Solución (para el instructor)

1. Script en `/usr/local/bin` (lo ejecutará systemd) con copia en `~/bin`:

```bash
cat > /tmp/limpiar-logs.sh <<'EOF'
#!/bin/bash
# limpiar-logs.sh - comprime .log antiguos y elimina .gz de más de 30 días
# Uso:    limpiar-logs.sh DIRECTORIO DIAS
# Salida: 0 ok · 1 argumentos inválidos
set -euo pipefail

DIR="${1:-}"
DIAS="${2:-}"
RETENCION_GZ=30
TAG="limpiar-logs"

if [[ -z "$DIR" || -z "$DIAS" ]]; then
    echo "Uso: $0 DIRECTORIO DIAS" >&2
    exit 1
fi
if [[ ! -d "$DIR" ]]; then
    echo "ERROR: el directorio $DIR no existe" >&2
    logger -t "$TAG" "ERROR: el directorio $DIR no existe"
    exit 1
fi
if [[ ! "$DIAS" =~ ^[0-9]+$ ]]; then
    echo "ERROR: DIAS debe ser un número entero, se recibió '$DIAS'" >&2
    exit 1
fi

comprimidos=0
while IFS= read -r -d '' archivo; do
    gzip "$archivo"
    comprimidos=$((comprimidos + 1))
done < <(find "$DIR" -maxdepth 1 -type f -name "*.log" -mtime +"$DIAS" -print0)

borrados=$(find "$DIR" -maxdepth 1 -type f -name "*.gz" -mtime +"$RETENCION_GZ" -print -delete | wc -l)

logger -t "$TAG" "$DIR: $comprimidos .log comprimidos (> $DIAS días), $borrados .gz eliminados (> $RETENCION_GZ días)"
echo "$(date '+%F %T') $DIR: $comprimidos comprimidos, $borrados eliminados"
EOF
sudo cp /tmp/limpiar-logs.sh /usr/local/bin/limpiar-logs.sh
sudo chmod 755 /usr/local/bin/limpiar-logs.sh
cp /tmp/limpiar-logs.sh ~/bin/
chmod +x ~/bin/limpiar-logs.sh
bash -n ~/bin/limpiar-logs.sh && echo "sintaxis ok"
```
Qué observar: el `chmod +x` sobre la copia de `~/bin` es obligatorio; `cat > archivo` crea el script con modo 644 y `cp` conserva ese modo, así que sin él el paso 2 fallaría con `Permission denied`.

2. Pruebas de validación y ejecución real:

```bash
limpiar-logs.sh; echo "código: $?"
limpiar-logs.sh /no/existe 7; echo "código: $?"
limpiar-logs.sh ~/empresa/logs/app siete; echo "código: $?"
limpiar-logs.sh ~/empresa/logs/app 7
ls ~/empresa/logs/app
sudo journalctl -t limpiar-logs -n 1 --no-pager
```

Salida esperada:
```
Uso: /home/student/bin/limpiar-logs.sh DIRECTORIO DIAS
código: 1
ERROR: el directorio /no/existe no existe
código: 1
ERROR: DIAS debe ser un número entero, se recibió 'siete'
código: 1
2026-09-03 11:02:35 /home/student/empresa/logs/app: 5 comprimidos, 3 eliminados
reciente-1.log  reciente-3.log  reciente-5.log  viejo-2.log.gz  viejo-4.log.gz
reciente-2.log  reciente-4.log  viejo-1.log.gz  viejo-3.log.gz  viejo-5.log.gz
Sep 03 11:02:35 rhel01 limpiar-logs[7420]: /home/student/empresa/logs/app: 5 .log comprimidos (> 7 días), 3 .gz eliminados (> 30 días)
```
Detalle que conviene comentar: `gzip` conserva la fecha de modificación del original, así que `viejo-*.log.gz` sigue teniendo 10 días y no se borra hasta cumplir 30; un `.log` de 40 días se comprimiría y se borraría en la misma ejecución.

3. Programación con cron (a las 02:00, lunes a viernes) en el crontab de `student`:

```bash
(crontab -l; echo "0 2 * * 1-5 /home/student/bin/limpiar-logs.sh /home/student/empresa/logs/app 7 >> /home/student/limpiar-logs.log 2>&1") | crontab -
crontab -l | tail -1
```

Salida esperada:
```
0 2 * * 1-5 /home/student/bin/limpiar-logs.sh /home/student/empresa/logs/app 7 >> /home/student/limpiar-logs.log 2>&1
```

4. Programación equivalente con systemd timer (el servicio corre como `student` para que los archivos sigan siendo suyos):

```bash
sudo tee /etc/systemd/system/limpiar-logs.service > /dev/null <<'EOF'
[Unit]
Description=Compresión y limpieza de logs de la aplicación de expedientes (PGN)

[Service]
Type=oneshot
User=student
ExecStart=/usr/local/bin/limpiar-logs.sh /home/student/empresa/logs/app 7
EOF
sudo tee /etc/systemd/system/limpiar-logs.timer > /dev/null <<'EOF'
[Unit]
Description=Ejecuta limpiar-logs de lunes a viernes a las 02:00

[Timer]
OnCalendar=Mon..Fri 02:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now limpiar-logs.timer
systemd-analyze calendar "Mon..Fri 02:00" | grep Next
systemctl list-timers limpiar-logs.timer --no-pager | head -2
sudo systemctl start limpiar-logs.service && sudo journalctl -u limpiar-logs.service -n 4 --no-pager
```

Salida esperada:
```
    Next elapse: Fri 2026-09-04 02:00:00 EST
NEXT                        LEFT     LAST PASSED UNIT               ACTIVATES
Fri 2026-09-04 02:00:00 EST 14h left n/a  n/a    limpiar-logs.timer limpiar-logs.service
Sep 03 11:08:10 rhel01 systemd[1]: Starting Compresión y limpieza de logs de la aplicación de expedientes (PGN)...
Sep 03 11:08:10 rhel01 limpiar-logs.sh[7602]: 2026-09-03 11:08:10 /home/student/empresa/logs/app: 0 comprimidos, 0 eliminados
Sep 03 11:08:10 rhel01 systemd[1]: limpiar-logs.service: Deactivated successfully.
Sep 03 11:08:10 rhel01 systemd[1]: Finished Compresión y limpieza de logs de la aplicación de expedientes (PGN).
```
Qué observar: systemd imprime literalmente el texto de `Description=` en las líneas `Starting…` y `Finished…` (por eso conviene que sea corto). La ejecución da 0 y 0 porque el paso 2 ya comprimió y borró todo lo que había.
Cierre del reto: en producción se deja **una** de las dos programaciones, no ambas (se ejecutaría dos veces). Pregunta para la puesta en común: ¿cuál elegirían y por qué? (Timer: journal integrado, `Persistent`, `User=`; cron: portable y conocido por todos.)

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `#!/bin/bash` + `chmod +x script.sh` | Declarar el intérprete y permitir ejecutar `./script.sh` |
| `bash -n script.sh` / `bash -x script.sh` | Revisar sintaxis / ejecutar mostrando cada comando expandido |
| `set -euo pipefail` | Modo estricto: parar en errores, variables no definidas y fallos en tuberías |
| `VAR="valor"; echo "${VAR}"` | Asignar (sin espacios) y leer una variable |
| `$0 $1 $# "$@" $? $$` | Nombre del script, primer argumento, cantidad, todos, último exit code, PID |
| `"${1:-/valor/por/defecto}"` | Argumento con valor por defecto |
| `FECHA=$(date +%F)` / `$(( 5 + 3 ))` | Sustitución de comandos / aritmética entera |
| `read -p "Texto: " var` | Leer del teclado |
| `[[ -d "$d" ]]`, `[[ -f "$f" ]]`, `[[ -z "$s" ]]`, `[[ $a -gt $b ]]` | Pruebas de directorio, archivo, cadena vacía, comparación numérica |
| `if … then … elif … else … fi` / `case "$x" in a) ;; *) ;; esac` | Condicionales |
| `for f in *.log; do …; done` / `for i in {1..10}` | Bucles sobre archivos y secuencias |
| `while IFS=, read -r a b c; do …; done < archivo.csv` | Procesar un archivo línea a línea con campos |
| `done < <(comando)` | Bucle sobre la salida de un comando sin perder variables (evita la subshell) |
| `nombre() { local x="$1"; …; return 0; }` | Definir una función |
| `arr=(a b c); "${arr[@]}"; ${#arr[@]}` | Arrays: definir, recorrer, contar |
| `awk '{print $1}'` / `awk -F: '$3 >= 1000 {print $1}'` | Extraer columnas y filtrar por condición |
| `sed -i.bak 's/viejo/nuevo/g' archivo` | Sustituir en el archivo con copia de respaldo |
| `sort \| uniq -c \| sort -rn` | Contar repeticiones y ordenar por frecuencia |
| `logger -t etiqueta -p local0.warning "msg"` | Registrar un mensaje en el journal y en /var/log/messages |
| `crontab -e` / `-l` / `-r` / `sudo crontab -u ana -l` | Editar, listar, borrar el crontab; ver el de otro usuario |
| `*/15 8-17 * * 1-5 comando >> log 2>&1` | Cada 15 min, de 8 a 17 h, lunes a viernes, con salida redirigida |
| `/etc/crontab`, `/etc/cron.d/`, `/etc/cron.{hourly,daily,weekly,monthly}`, `/etc/anacrontab` | Programación del sistema y anacron |
| `sudo tail /var/log/cron` / `sudo journalctl -u crond` | Ver qué ejecutó cron |
| `at now + 5 minutes` / `atq` / `at -c N` / `atrm N` / `batch` | Trabajos únicos con at |
| `systemctl list-timers --all` | Ver todos los timers y sus próximas ejecuciones |
| `systemctl enable --now nombre.timer` | Activar un timer (tras `daemon-reload`) |
| `systemd-analyze calendar "Mon..Fri 02:00"` | Comprobar una expresión `OnCalendar` |
| `systemd-tmpfiles --create` / `--clean archivo.conf` | Aplicar reglas de `/etc/tmpfiles.d/` |
| `ansible -i inventario all -m ping` / `ansible-playbook -i inventario play.yml -K` | Probar conectividad y ejecutar un playbook con sudo |
| `systemctl enable --now cockpit.socket` | Activar la consola web en el puerto 9090 |

---

## Notas para el instructor

- **Preparar antes de la clase:**
  - Ejecutar **todos** los labs en orden en la VM del instructor la noche anterior, partiendo del snapshot `dia06-fin`. Guardar los scripts (`backup.sh`, `crear-usuarios.sh`, `healthcheck.sh`, `monitor-disco.sh`, `limpiar-logs.sh`), `usuarios.csv`, los unit files y los generadores de datos en un solo archivo de texto para pegarlos en el chat por bloques: en 4 horas no hay tiempo para que los participantes tecleen 200 líneas.
  - Confirmar con `sudo dnf install -y at ansible-core cockpit dos2unix tree` que los repositorios responden (`subscription-manager status`); anotar el tamaño de la descarga de `ansible-core` para prever el tiempo con la red del aula.
  - Verificar `rpm -q cronie cronie-anacron at cockpit` para saber qué viene instalado y qué no, y decirlo al empezar el Bloque 4.
  - Agregar la regla de port forwarding **9090 → 9090** en la VM del instructor para la demo de Cockpit.
  - Comprobar que `/datos` está montado con `findmnt /datos` (no con `df -h /datos`, que siempre responde aunque sea con los datos de `/`); si el Día 6 quedó a medias, tener listo el comando de fallback del Lab 1.1 paso 1.
  - Revisar que `ana`, `carlos`, `pedro` y `laura` y los grupos `sistemas`, `soporte`, `auditoria` existen (Día 3): `getent group sistemas soporte auditoria`. El Lab 4.1 usa `jperez` del Lab 3.1 para no depender de ellos.
  - ⚠️ Verificar en la VM: (a) que `sudo -u jperez crontab -l` funciona tras `chage -d "$(date +%F)" jperez` (Lab 4.1 paso 6, PAM); (b) cuántas líneas finales muestra `at -c N` para ajustar el `tail` del Lab 4.2; (c) que `systemctl start monitor-disco.service` no da `203/EXEC` con SELinux enforcing (el script se copió con `cp`, contexto `bin_t`); (d) la versión real de `ansible-core` (`ansible --version`) y de `cockpit` (`rpm -q cockpit`) para corregir las salidas esperadas del Bloque 5; (e) los UID/GID que asigna `useradd` a `lmartinez`…`mcastro` (Lab 3.1) y cuántos filesystems cuenta `monitor-disco.sh` en esa VM (3 en VirtualBox, 4 en UTM con `/boot/efi`); (f) si `cockpit.socket` ya está habilitado desde la instalación, para no prometer la línea `Created symlink…`.
  - Tener a mano el snapshot `dia06-fin` para restaurar la VM de algún participante en 2 minutos.

- **Qué estudiar si es nuevo en RHEL:**
  1. **systemd timers**: `man systemd.timer`, `man systemd.time` (sección CALENDAR EVENTS) y practicar `systemd-analyze calendar` con 5 expresiones distintas; ver `systemctl cat dnf-makecache.timer` y `logrotate.timer` como ejemplos reales de RHEL 9.
  2. **cron en RHEL**: `man 5 crontab`, `man anacrontab`, leer `/etc/cron.d/0hourly` y `/etc/cron.hourly/0anacron` para poder explicar la cadena crond → run-parts → anacron. Confirmar en la VM el mensaje exacto de `command not found` de una tarea de cron y que aparece en `/var/log/cron`.
  3. **systemd-tmpfiles**: `man tmpfiles.d`, leer `/usr/lib/tmpfiles.d/tmp.conf` y probar `systemd-tmpfiles --create` sobre un `.conf` propio. Tener clara la explicación de por qué `touch -d` no sirve para simular la limpieza (ctime).
  4. **SELinux y scripts ejecutados por systemd**: `ls -Z` de un script copiado con `cp` (bin_t) vs movido con `mv` desde `/home` (user_home_t); probar que `restorecon -v` lo corrige. Si un `ExecStart` falla con `status=203/EXEC`, casi siempre es permiso `x` o contexto.
  5. **ansible-core en RHEL 9**: `ansible-doc ansible.builtin.dnf`, `ansible-doc ansible.builtin.copy`; ejecutar el playbook del Lab 5.1 dos veces y una tercera con `-v` para ver el detalle. Saber responder "¿y para 10 servidores?": inventario con IPs, `ansible_user`, llave SSH.

- **Errores frecuentes de los participantes y cómo resolverlos:**

| Síntoma | Causa | Solución |
|---|---|---|
| `bash: ./hola.sh: Permission denied` | Falta el permiso de ejecución | `chmod +x hola.sh` (o ejecutar con `bash hola.sh`) |
| `hola.sh: command not found` a mano | El script no está en `~/bin` o la sesión no recargó el PATH | `ls ~/bin`, `echo $PATH`, `source ~/.bashrc`; o llamar con `./` |
| `/bin/bash^M: bad interpreter: No such file or directory` | Finales de línea CRLF (editado en Windows y pegado) | `sed -i 's/\r$//' script.sh` o `dos2unix script.sh`; ver con `cat -A` |
| `syntax error near unexpected token` o `[: missing ']'` | Falta espacio en `[ ]`, `then` en la misma línea sin `;`, `fi` olvidado | `bash -n script.sh`; recordar `if [[ … ]]; then` |
| `NOMBRE: command not found` al asignar | Espacios en la asignación `NOMBRE = valor` | `NOMBRE=valor` sin espacios |
| Funciona a mano, en cron dice `command not found` | PATH mínimo de cron | Rutas absolutas o `PATH=` al inicio del crontab |
| Cron no ejecuta nada y no hay nada en `/var/log/cron` | `crond` inactivo, usuario en `cron.deny` o crontab sin salto de línea final | `systemctl status crond`, `cat /etc/cron.deny`, `crontab -l` |
| `sudo: crear-usuarios.sh: command not found` | `sudo` usa `secure_path`, no el PATH del usuario | `sudo ~/bin/crear-usuarios.sh …` (ruta completa) |
| `You (usuario) are not allowed to use this program (crontab)` | Usuario listado en `/etc/cron.deny` o no listado en `cron.allow` | Editar esos archivos |
| `You (usuario) are not allowed to access to (crontab) because of pam configuration` | La cuenta tiene la contraseña vencida o el cambio forzado (`chage -d 0`) y PAM (`/etc/pam.d/crond`) lo rechaza | `sudo chage -d "$(date +%F)" usuario` o que el usuario cambie su contraseña; ver `sudo chage -l usuario` |
| `at: command not found` o `Can't open /run/atd.pid to signal atd` | Paquete `at` no instalado / `atd` inactivo | `sudo dnf install -y at; sudo systemctl enable --now atd` |
| El timer no aparece en `list-timers` | Falta `daemon-reload`, `enable --now`, o el `[Install]` en el `.timer` | `sudo systemctl daemon-reload && sudo systemctl enable --now nombre.timer` |
| `status=203/EXEC` en `journalctl -u nombre.service` | Script sin permiso `x` o contexto SELinux `user_home_t` (movido con `mv`) | `sudo chmod 755 …`, `sudo restorecon -v /usr/local/bin/script.sh` |
| El script termina sin mensaje al llegar a `((i++))` | Con `set -e`, `((0++))` devuelve 1 | Usar `i=$((i+1))` |
| `unbound variable` | `set -u` y variable sin definir (a menudo `$1` ausente) | `"${1:-valor}"` |
| El contador dentro del `while` siempre da 0 | `comando \| while` ejecuta el bucle en una subshell | `done < <(comando)` |
| El `while read` salta la última línea del CSV | El archivo no termina en salto de línea | Agregar la línea final (los here-doc la incluyen) |
| Ansible: `Missing sudo password` | `become: true` sin `-K` | `ansible-playbook … -K` |
| Ansible: `Failed to download metadata` / `No package tree available` | Sistema sin suscripción o sin red | `sudo subscription-manager status`, `dnf repolist` |
| Cockpit no carga en `https://localhost:9090` | Falta la regla de port forwarding 9090 o el socket está inactivo | Agregar la regla; `systemctl status cockpit.socket` |
| `crontab -r` ejecutado por error | Se borró el crontab completo | No se recupera; volver a instalarlo desde `~/mi-crontab` (por eso se guarda en un archivo) |

- **Diferencias VirtualBox (x86_64) vs UTM (aarch64):**
  - Ningún comando del día cambia. Las diferencias visibles: `df` y `/var/log/monitor-disco.log` muestran las particiones como `/dev/sda*` en VirtualBox y `/dev/vda*` en UTM (los volúmenes lógicos se ven igual en ambos: `/dev/mapper/rhel-root`, `/dev/mapper/vg_datos-lv_datos`); `uname -r` y los nombres de paquetes terminan en `x86_64` o `aarch64`; en UTM (arranque UEFI) `df` lista además `/boot/efi` y puede listar `efivarfs`, por eso el script lo excluye y por eso "Revisados" puede dar 4 en lugar de 3.
  - La regla de port forwarding 9090 se configura en lugares distintos (VirtualBox: Adaptador NAT → Avanzadas → Reenvío de puertos; UTM: configuración del dispositivo de red en modo "Emulated VLAN", que es el que ya usan para 2222). ⚠️ Verificar en la VM antes de la clase; si da problemas, usar el túnel SSH `ssh -p 2222 -L 9090:localhost:9090 -N student@localhost`, que funciona igual en los dos hipervisores.
  - En Ansible, `ansible_architecture` dirá `x86_64` o `aarch64`; si algún participante ve `arm64`, no es RHEL: es otra imagen.

- **Preguntas probables y respuesta corta:**
  - *¿Cron o systemd timer? ¿Cuál debo usar?* Para tareas de un usuario o de una aplicación, cron sigue siendo lo habitual y lo que pide el RHCSA. Para tareas del sistema que necesitan logs en el journal, dependencias (`After=network-online.target`), `User=` o recuperar ejecuciones perdidas, timer. Ambas son válidas; lo que no se hace es programar la misma tarea en las dos.
  - *¿Por qué no me llega el correo de cron?* RHEL 9 no instala un servidor de correo; la salida se descarta. Redirigir a un archivo con `>> log 2>&1` o usar `logger`.
  - *Si el servidor está apagado a las 02:00, ¿se ejecuta después?* Con cron de usuario, no (se pierde). Con anacron (daily/weekly/monthly del sistema), sí, al encender. Con un timer y `Persistent=true`, sí, al arrancar.
  - *¿Puedo usar Python en lugar de Bash?* Sí (`#!/usr/bin/python3`, RHEL 9 lo incluye), y cron o systemd lo ejecutan igual. Bash sigue siendo el idioma común de la administración y del examen.
  - *¿Cómo evito que dos ejecuciones se pisen si la tarea dura más que el intervalo?* `flock -n /tmp/tarea.lock comando` en el crontab (root puede usar `/run/lock/tarea.lock`; un usuario normal no tiene permiso de escritura en `/run/lock`); los timers de systemd no lanzan el servicio si aún está corriendo.
  - *¿Ansible necesita instalar un agente en los servidores?* No: SSH y Python, que RHEL ya tiene. Solo el equipo que ejecuta Ansible (nodo de control) necesita `ansible-core`.
  - *¿Puedo ver los timers de otro usuario o crear los míos sin root?* Sí: `systemctl --user` permite timers por usuario en `~/.config/systemd/user/`; en este curso se trabaja con los del sistema.
  - *¿`sh` y `bash` son lo mismo?* En RHEL, `/bin/sh` es bash ejecutándose en modo compatible POSIX: `[[ ]]`, arrays y `{1..10}` pueden no funcionar. Por eso el shebang debe decir `bash` y cron define `SHELL=/bin/bash` si el comando lo necesita.

- **Relación con el examen RHCSA (EX200):**
  - Objetivo "Create simple shell scripts": ejecutar código condicionalmente (`if`, `test`, `[ ]`), bucles (`for`) para procesar archivos y entradas de línea de comandos, procesar argumentos (`$1`, `$2`), procesar la salida de comandos dentro de un script. Todo el Bloque 1 y 2 apunta ahí; en el examen se piden scripts cortos (10–20 líneas), no programas.
  - Objetivo "Schedule tasks using at and cron": `crontab -e` para un usuario concreto (`sudo crontab -u usuario -e`), formato de los 5 campos, `at`. Es habitual un enunciado como "el usuario X debe ejecutar `logger` cada día a las 14:23 de lunes a viernes".
  - systemd timers y `systemd-tmpfiles` no figuran literalmente en la lista pública de objetivos de EX200, pero sí en el temario oficial de RH134 ("Schedule Future Tasks", "Manage Temporary Files") y conviene dominarlos: saber leer `systemctl list-timers` y `/etc/tmpfiles.d/` ayuda a entender el sistema que aparece en el examen.
  - Ansible **no** está en EX200 (es EX294/RHCE); Cockpit tampoco. Se ven como contexto profesional.

## Tarea y preparación para el día siguiente

- Tomar el snapshot **`dia07-fin`** con la VM apagada. Dejar activos `crond`, `atd`, `monitor-disco.timer` y `limpiar-logs.timer`: mañana se revisa `sudo journalctl -u monitor-disco.service --since today | tail` para ver las ejecuciones nocturnas (y, si la VM estuvo apagada, el efecto de `Persistent=true`).
- Práctica de 20 minutos: escribir `~/bin/usuarios-sin-shell.sh` que recorra `/etc/passwd` con `while IFS=: read -r …` e imprima los usuarios cuyo UID sea mayor o igual a 1000 y cuya shell sea `/bin/bash`, con el formato `usuario -> shell`; programarlo con `at` para dentro de 3 minutos redirigiendo la salida a `~/usuarios.txt`. Sin pistas adicionales; se comenta al inicio del Día 8.
- Quien no completó el reto: hacerlo siguiendo el enunciado (la solución se publica mañana).
- Agregar la regla de port forwarding 9090 → 9090 y activar Cockpit (`sudo systemctl enable --now cockpit.socket`) para explorarlo 10 minutos.
- Lecturas: `man 5 crontab` (sección de ejemplos), `man systemd.time` (sección CALENDAR EVENTS) y `man tmpfiles.d` (solo el formato de línea).
- Comprobar con `nmcli device status` que la VM tiene ambas interfaces (NAT y host-only) en estado `connected`: los próximos días se trabaja con servicios de red y se necesita la IP host-only.
- Limpieza opcional si quieren dejar la VM más ordenada (no obligatorio; el snapshot lo conserva todo): `crontab -l > ~/crontab-dia07.txt` como respaldo del crontab; los usuarios `lmartinez`, `rgomez`, `jperez` y `mcastro` se pueden conservar para los ejercicios de troubleshooting.
