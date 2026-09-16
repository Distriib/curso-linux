# 2 — Archivos, directorios y enlaces (guía del instructor)

Igual que el Bloque 1: tipeás mientras explicás. Todo el demo pasa en
`/tmp/demo` para no pisar nada; al final se borra. Lo importante: que al
terminar tengan claro `cp`/`mv`/`rm`, la diferencia duro/simbólico y que
`*` lo expande la shell — con eso hacen el lab solos.

## Leer `ls -l`

```bash
ls -l /etc/hostname
```
**Qué decir:** "esta línea la van a leer mil veces; vamos columna por
columna".
**Qué señalar:** de izquierda a derecha — tipo, permisos (Día 3, no
profundizar), el punto (SELinux, Día 8), número de enlaces duros (vuelve
en la sección de enlaces), dueño, grupo, tamaño, fecha, nombre.

tipo de archivo :
"d" = directorio

### Tipos de archivo

```bash
ls -l /etc/hostname /bin /dev/null /dev/sda
```
**Qué señalar:** `-`, `l` (con `->`), `c`, `b`. En UTM es `/dev/vda`.
Frase: "en Linux todo es un archivo: un disco, la terminal, la nada
(`/dev/null`)". Los dispositivos muestran dos números (`1, 3`) en vez de
tamaño — solo mencionarlo.

### Opciones de `ls`

No hace falta demostrar cada una; la tabla queda de referencia. Si querés
una: `ls -lt /etc | head -3` — "lo último que se tocó en `/etc`".

El inodo es el número de identidad que el disco le da a cada archivo. Los datos y sus metadatos (dueño, permisos, fechas) viven en el inodo; el nombre es solo una etiqueta que apunta a ese número.
ls -li /etc/hostname
---

## Crear, copiar, mover, borrar

```bash
cd /tmp
mkdir -p demo/a/b
touch demo/x.txt
echo "hola" > demo/x.txt
cp demo/x.txt demo/y.txt
mv demo/y.txt demo/z.txt
ls -l demo
```
**Qué decir:** "`mkdir -p` crea toda la cadena de una; `touch` crea
vacío; `cp` copia; `mv` mueve **y** renombra — para Linux es lo mismo".
**Qué señalar:** después del `mv`, `y.txt` ya no existe, hay `z.txt`.

Dos trampas que conviene decir en voz alta (no demostrar, solo avisar):
- `cp -r dir destino`: si `destino` existe, crea `destino/dir`; si no
  existe, crea `destino` como copia. Van a caer en esto en el lab.
- `cp` normal pone fecha de hoy y dueño quien copia; `cp -a` conserva
  fechas, permisos y dueño. Para respaldos, `-a`.

```bash
rm -i demo/z.txt
rmdir demo/a
rm -r demo/a
ls demo
```
**Qué decir:** "no hay papelera. `rm -i` pregunta; `rmdir` solo borra
vacíos — falla con `a` porque tiene `b` adentro; `rm -r` sí lo borra".
**Qué señalar:** el `rmdir: failed ... Directory not empty` es
esperado. Decir completo, mirándolos: **"`rm -rf` con una ruta mal
escrita o un espacio de más ha borrado servidores en producción. Hábito:
`ls` de la ruta antes del `rm -r`."**

---

## Enlaces

```bash
ln demo/x.txt demo/x-duro.txt
ln -s /tmp/demo/x.txt demo/x-link.txt
ls -li demo
```
**Qué decir:** "un archivo es un inodo (los datos) más uno o más nombres. El duro es otro nombre para el mismo inodo; el simbólico es un
archivito que contiene una ruta".
**Qué señalar:** `x.txt` y `x-duro.txt` tienen el **mismo número de
inodo** (primera columna) y un `2` en la columna de enlaces. `x-link.txt`
tiene inodo propio, tipo `l`, y muestra `-> /tmp/demo/x.txt`. El
simbólico se crea con **ruta absoluta** — si lo hacen con relativa desde
otra carpeta, queda roto; es el error #1 del lab.

```bash
rm demo/x.txt
cat demo/x-duro.txt
cat demo/x-link.txt
```
**Qué decir:** "borro el original y miro cuál sobrevive".
**Qué señalar:** el duro sigue mostrando `hola`; el simbólico dice `No
such file or directory` — quedó apuntando a algo que ya no está (con
`ls --color` se ve en rojo). En la práctica se usa casi siempre el
simbólico: `/bin -> usr/bin`, "último backup", versiones de software.

---

## Globbing

```bash
echo /etc/host*
ls /etc/*.conf | wc -l
touch demo/f{1..3}.txt
ls demo/f?.txt
echo demo/*
rm -r demo
cd
```
**Qué decir:** "el `*` no lo entiende `ls`, lo expande la shell **antes**
de llamar a `ls`. Por eso `echo patrón` te muestra exactamente lo que va
a recibir cualquier comando — es la forma de probar antes de un `rm`".
**Qué señalar:** `{1..3}` generó tres archivos con un solo `touch`; `?`
es exactamente un carácter. Los ocultos (`.algo`) no entran en `*`.

Cerrar con: "con esto tienen todo para el lab: `mkdir -p`, `touch`,
`echo >`, `cp`, `mv`, `ln`, `ln -s`, `rm`, `tree`. Ahora lo hacen solos."

wc word count
-l cuantas lineas 