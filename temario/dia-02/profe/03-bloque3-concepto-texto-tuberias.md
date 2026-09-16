# 3 — Texto, tuberías y redirección (guía del instructor)

Tipeás mientras explicás. Todo el demo usa `/etc/passwd` (lo puede leer cualquiera, tiene un usuario por línea con campos separados por `:`) y `/tmp` para lo que se escribe. Lo que tienen que sacar de acá para el
lab: `grep` con `-c -v -n -w -E`, el patrón `cut | sort | uniq -c | sort -rn`,
y `>` / `>>` / `2>` / `|`.

/etc/passwd no tiene contraseñas — el nombre es histórico, las tuvo hace 30 años. Hoy es la lista de cuentas de usuario, una por línea, con 7 campos separados por ::


student:x:1000:1000:student:/home/student:/bin/bash
   1    2   3    4     5          6            7
1 nombre · 2 x (la contraseña está en otro lado) · 3 UID · 4 GID · 5 descripción · 6 home · 7 shell

## Ver texto

```bash
head -3 /etc/passwd
tail -3 /etc/passwd
wc -l /etc/passwd
less /etc/services
```
**Qué decir:** "`/etc/passwd` es la lista de usuarios del sistema, uno por línea. `head` y `tail` son las primeras y las últimas líneas; `wc -l`
cuenta cuántas hay".
**Qué señalar:** `wc -l` da unos 40 y pico — hay muchos más usuarios que `student` y `root`: son usuarios de servicios (se ve el Día 3). En `less` probá `/ssh`, `n` un par de veces, y `q`. Decir: "`man` usa `less`, así que esto mismo sirve para los manuales". Mencionar `tail -f` sin demostrarlo todavía — se hace en el lab.

head / tail / wc -l / less — primeras líneas / últimas / contar líneas / abrir para recorrer (como man).
---

## `grep`

```bash
grep student /etc/passwd
grep -n bash /etc/passwd
grep -c nologin /etc/passwd
grep -v nologin /etc/passwd
grep -i STUDENT /etc/passwd
```
**Qué decir:** "`grep` busca líneas que contengan algo y te las muestra.
Es el comando que más van a usar en soporte".
**Qué señalar:** `-n` pone el número de línea adelante; `-c` solo da la cantidad, no las líneas; `-v` invierte ("los que **no** tienen
`nologin`" = los que sí pueden entrar); `-i` encuentra `student` aunque
lo escriban en mayúsculas.

`nologin` no hace falta explicarlo en profundidad: "es la shell que se le
pone a un usuario que no debe poder iniciar sesión, Día 3".

### Expresiones regulares

```bash
grep -E "^student|^root" /etc/passwd
grep -E "bash$" /etc/passwd
grep -Ev "^#|^$" /etc/ssh/sshd_config
```
**Qué decir:** "con `-E` el patrón puede tener símbolos especiales: `^`
es 'empieza con', `$` es 'termina con', `|` es 'o'".
**Qué señalar:** el primero trae solo las líneas que **empiezan** con
`student` o `root` (sin `^` traería también las que lo tengan en el medio). El último es el que más se usa en la vida real: **"mostrame la
configuración sin comentarios (`^#`) ni líneas vacías (`^$`)"** — quedan tres o cuatro líneas de un archivo de más de cien. Frase para que se la
lleven: "`grep -Ev "^#|^$"` sobre cualquier `.conf` = lo que está activo".

En el teclado latinoamericano `|` es `AltGr + 1` (o cerca); que lo prueben ahora si no lo hicieron.

---

## Cortar, ordenar, contar

```bash
cut -d: -f1 /etc/passwd
cut -d: -f7 /etc/passwd | sort
cut -d: -f7 /etc/passwd | sort | uniq -c
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```
**Qué decir:** "`cut` corta cada línea en pedazos usando un separador y
te da el pedazo que pidas. En `/etc/passwd` el separador es `:` y el
campo 7 es la shell del usuario".
**Qué señalar:** construí la tubería **de a un comando por vez**, para
que vean cómo cada `|` transforma la salida anterior:
1. `cut` → una shell por línea, desordenadas.
2. `| sort` → las mismas, ordenadas, las repetidas quedan juntas.
3. `| uniq -c` → cada shell una vez, con cuántas veces aparecía.
4. `| sort -rn` → las más repetidas arriba.

Resultado: casi todos los usuarios tienen `/sbin/nologin`, dos o tres
tienen `/bin/bash`. Decir: **"`sort | uniq -c | sort -rn` es 'el top de lo
más repetido'; lo van a usar en el lab con el log"**. Y el porqué del
`sort` antes de `uniq`: `uniq` solo junta repetidos que están **pegados**;
si no ordenás antes, cuenta mal.

---

## Los tres flujos y la redirección

**Qué decir:** "cada programa tiene tres cables: uno por donde lee, uno
por donde responde, y uno aparte por donde se queja. Normalmente los
tres van a la pantalla, pero se pueden desviar".

```bash
cd /tmp
grep bash /etc/passwd > shells.txt
grep nologin /etc/passwd >> shells.txt
wc -l shells.txt
```
**Qué señalar:** el primer `grep` no mostró nada en pantalla — fue al
archivo. `>` crea o **pisa**; `>>` agrega. `wc -l` confirma que tiene
las dos cosas.

```bash
ls /etc/hostname /noexiste
ls /etc/hostname /noexiste > ok.txt 2> error.txt
cat ok.txt
cat error.txt
ls /noexiste 2> /dev/null
```
**Qué señalar:** el primer `ls` muestra en pantalla la respuesta y la
queja mezcladas. El segundo las separa: `>` se lleva la respuesta, `2>` se lleva la queja. `2>
/dev/null` = "la queja tirala" — `/dev/null` es el agujero negro. Si
alguien pregunta por qué `2`: "la salida normal es la 1, los errores son
la 2; `>` solo es `1>` abreviado".

```bash
grep bash /etc/passwd | tee bash.txt | wc -l
```
**Qué decir:** "`tee` es una T de plomería: guarda una copia en el archivo
y deja seguir el flujo". **Qué señalar:** se creó `bash.txt` **y** `wc`
recibió las líneas igual.

### `sudo` y `>` — el que engaña a todos

/etc/motd = message of the day. Es un archivo de texto que el sistema muestra automáticamente a cualquiera que inicie sesión (por SSH o en la consola), justo después de la contraseña y antes del prompt. En una instalación nueva está vacío.

```bash
sudo echo "Servidor rhel01 - PGN" > /etc/motd
echo "Servidor rhel01 - PGN" | sudo tee /etc/motd
cat /etc/motd
```
**Qué decir:** "esto lo van a intentar el primer día que administren un servidor, y va a fallar".
**Qué señalar:** el primero da `Permission denied` aunque tenga `sudo`.
Por qué: el `>` lo hace **la shell de `student`**, antes de que `sudo` entre en acción; `sudo` solo le dio permiso al `echo`, no a la
redirección. La forma correcta es pasar el texto por tubería a `sudo tee`. El `/etc/motd` es el mensaje del día — les va a aparecer la próxima vez que entren por SSH.

```bash
rm shells.txt ok.txt error.txt bash.txt
cd
```
Limpieza. Cerrar con: "en el lab van a hacer esto mismo sobre un log de
300 líneas: contar, filtrar, sacar tops y guardar resultados".
