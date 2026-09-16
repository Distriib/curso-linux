# 1 — Cómo funciona la shell (guía del instructor)

Esto es puro concepto, sin lab — el Lab 1.1 (siguiente archivo) es donde
ellos practican cada una de estas cosas escribiendo. Acá solo tenés que
leer/mostrar rápido y pasar; no hace falta demostrar nada en consola todavía,
eso pasa en el lab.

## El prompt

`student@rhel01 ~]$` — decí: la shell es el programa que lee lo que
escriben y lo ejecuta (bash, en RHEL). El prompt es la señal de "lista para
recibir un comando". Si el `$` cambia a `#`, están como root — remarcarlo,
es un dato que se repite todo el curso.

## Anatomía de un comando

Nada nuevo respecto al Día 1, es solo ponerle nombre a lo que ya vienen haciendo: comando, opciones, argumentos. Rápido.

-l listado largo
-a archivos ocultos
-h human readable

/etc es donde vive toda la configuración del sistema y de los servicios instalados, en archivos de texto plano (los podés leer con cat o less, y editar con vim, como van a hacer en el Lab 6.1 de hoy).


## Rutas

Ya lo vieron el Día 1 (`cd`, `pwd`, `~`). Lo único nuevo hoy es `cd -`
(volver al directorio anterior) — mencionarlo y seguir, se practica en el
lab.
~ atajo del home

## Variables de entorno — esto sí es nuevo, vale la pena pausar un poco más

**La idea antes de las variables puntuales:** una variable es una cajita con
nombre que guarda un pedacito de texto adentro. `echo $NOMBRE` muestra lo que
hay adentro de esa cajita (`echo` de por sí solo imprime texto en pantalla).

- **`$HOME`** → adentro tiene guardada la ruta de tu home (`/home/student`).
  Es lo mismo a lo que apunta el `~`, solo que como variable.
- **`$PATH`** → adentro tiene guardada una lista de carpetas (separadas por
  `:`). Cuando escribís `ls`, la shell no sabe mágicamente dónde está el
  programa — busca, una por una, en esas carpetas de la lista, hasta
  encontrarlo.
- **`$PS1`** → adentro tiene guardado el "diseño" del prompt: el texto
  `[\u@\h \W]\$` que, procesado, se convierte en `[student@rhel01 ~]$`. No
  hace falta explicar la sintaxis rara (`\u`, `\h`), solo que eso es lo que
  arma el prompt.
- **Mayúsculas**: no es obligatorio, es **convención** — las variables que
  ya trae el sistema se escriben en mayúsculas para distinguirlas de las
  que uno crea (`MIVAR=hola`, en minúscula).

Para hoy alcanza con que sepan **reconocerlas** con `echo $HOME`, `echo
$PATH`, `echo $PS1` — no es algo que vayan a usar activamente a diario,
salvo `$PATH` y `$?` (abajo), que sí aparecen seguido en troubleshooting.


- **`$?`** — cada comando, al terminar, devuelve un número diciendo si salió
  bien o mal; ese número queda guardado en `$?`. `0` = salió bien;
  cualquier otro número = algo falló. Se comprueba justo después del
  comando que te interesa (si corrés otro comando entre medio, `$?` ya es
  de ese otro, no del que querías revisar). Ejemplo para mostrar:
  ```bash
  ls /etc/hostname; echo $?     # existe → 0
  ls /no-existe; echo $?        # no existe → otro número (2)
  ```
  Se usará mucho en scripts más adelante (Día 7): ahí un script decide
  "sigo o me detengo" según este número.
- `export` — una variable creada a secas (`MIVAR=hola`) solo la ve esa
  shell; con `export` la heredan los programas que se lancen desde ahí.
  No hace falta profundizar hoy, alcanza con la idea general.

## Alias

RHEL ya trae varios definidos de fábrica (`ls`, `ll`, `grep` ya son alias con color). Mencionar que por eso `ls` sale con colores sin que nadie lo configure. `type comando` dice si algo es alias, builtin (parte de bash, como `cd`) o programa real.

type te devuelve una de tres respuestas, y cada una viene con su propia palabra clave — aliased to (es un apodo), is a shell builtin (viene con bash), o una ruta tipo /usr/bin/algo (es un programa real, y ahí está guardado).

Probalo:


type ls
type cd
type cat
Salida esperada:


ls is aliased to `ls --color=auto'
cd is a shell builtin
cat is /usr/bin/cat

## Historial

`!!` es el más útil de mostrar mentalmente: "me equivoco de permisos, en vez
de reescribir todo el comando pongo `sudo !!`". Ojo con un detalle: `!`
dentro de comillas dobles rompe (`echo "Hola!"` da error) — con comillas
simples no. Es un error frecuente, mencionarlo previene la pregunta después.

!$ ejemplo:

ls -l /etc/hostname
cat !$
!$ se reemplaza por /etc/hostname (el último argumento de la línea de ls), entonces termina corriendo cat /etc/hostname.

mkdir /tmp/carpeta-larga-de-escribir
cd !$

## Atajos de edición

No hace falta enseñar los ocho de memoria ahora — con mencionar 2 o 3
(`Ctrl+A`/`Ctrl+E` para moverse rápido, `Ctrl+L` para limpiar) alcanza; el
resto lo prueban solos en el lab y quedan en la tabla de referencia por si
los necesitan después.

## El árbol de RHEL 9

Repaso/ampliación de lo que ya vieron el Día 1 (ahí solo vieron el primer
nivel de `/`). Hoy se agrega el detalle de qué vive adentro de cada uno.
Dos datos que conviene tener a mano si preguntan:
- `/proc` y `/sys` ocupan 0 bytes en disco: son una ventana al estado del
  kernel en memoria, no archivos reales.
- `/bin`, `/sbin`, `/lib`, `/lib64` son enlaces simbólicos a sus
  equivalentes en `/usr` (desde RHEL 7, cambio llamado "UsrMove"); se ve
  con más detalle en el Lab 1.1, acá con nombrarlo alcanza.

No hay que agotar la tabla leyendo cada fila — es más una referencia para
que la vean en pantalla y usen mientras hacen el Lab 1.1 a continuación.
