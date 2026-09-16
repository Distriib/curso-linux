# 4 — Buscar archivos: `find` y `locate` (guía del instructor)

Tipeás mientras explicás. El demo usa `~/empresa` (lo conocen, saben qué
tiene que salir) y `/etc`. Diez minutos: `find` con tres criterios y
`-exec`, y `locate` con su instalación. Lo demás, tabla de referencia.

**Qué es `find`:** recorre una carpeta y todo lo que tiene adentro, y te
muestra lo que cumpla las condiciones que le des. Se escribe siempre en
el mismo orden: **dónde** buscar, **qué** condiciones, y opcionalmente
**qué hacer** con lo encontrado.

**Qué es `locate`:** busca un nombre de archivo en una lista que el sistema arma una vez por día. Instantáneo, pero no sabe de archivos creados después de la última lista.

---

## `find` — buscar recorriendo el disco

```bash
find ~/empresa -name "*.txt"
find ~/empresa -name "*.txt" | wc -l
find ~/empresa -type d
find ~/empresa -empty
```
**Qué decir:** "`find` es: dónde, qué, y opcionalmente qué hacer. Acá:
en `~/empresa`, los que terminan en `.txt`".
**Qué señalar:** salen los `.txt` de `documentos`, `clientes` y `logs` —
`find` **baja a todas las subcarpetas** solo, no como `ls`. Las comillas
en `"*.txt"` son obligatorias: sin ellas la shell expande el `*` antes
(globbing, Bloque 2) y `find` recibe otra cosa. `-type d` = solo
directorios: salen las 5 carpetas. `-empty` = vacíos: `backups`,
`plan-2026.doc`, `presupuesto.csv`.

### Criterios

No leer la tabla entera. Demostrar tres:

```bash
find /etc -name "*.conf"
sudo find /etc -name "*.conf" | wc -l
sudo find /etc -type f -size +100k
find ~/empresa -mmin -120
```
**Qué señalar:**
- El primero sin `sudo` mezcla resultados con varios `Permission denied`
  (hay carpetas de `/etc` que solo root lee). Con `sudo` se ve todo. La
  otra opción es `2> /dev/null` (Bloque 3) para tirar las quejas.
- `-size +100k` = "más grande que 100 KB". `+` es más que, `-` es menos
  que. Aparecen `/etc/services`, `/etc/udev/hwdb.bin` y poco más.
- `-mmin -120` = "modificado en los últimos 120 minutos". En `~/empresa`
  sale casi todo (lo hicieron hoy). **La pregunta real de soporte** es
  `sudo find /etc -mmin -120`: "¿qué cambió en la configuración en las
  últimas dos horas?" — ahí va a salir `/etc/motd`, que tocaron en el
  Bloque 3. Ese es el "ajá" del bloque.

### Acciones

**Qué es `-exec`:** en vez de solo mostrar lo encontrado, `find` ejecuta
un comando sobre cada resultado. `{}` es "el archivo encontrado", y hay
que terminar con `\;` o con `+`.

```bash
find ~/empresa -name "*.txt" -exec wc -l {} \;
find ~/empresa -name "*.txt" -exec wc -l {} +
```
**Qué decir:** "el `{}` es cada archivo que encontró; con `\;` corre `wc`
una vez por archivo, con `+` corre `wc` una sola vez con todos".
**Qué señalar:** el primero da una línea por archivo, sin `total`. El
segundo da lo mismo **más una línea `total`** — porque `wc` recibió
todos los archivos juntos. El `+` es más rápido; el `\;` sirve cuando el
comando solo acepta un archivo. La barra en `\;` es para que la shell no
se coma el punto y coma.

`-delete` no se demuestra en el concepto; se hace en el lab con una
carpeta de prueba. Solo decir: **"siempre primero el `find` sin
`-delete` para ver qué va a borrar, y después con"**.

---

## `locate`

```bash
sudo dnf install -y mlocate
locate servidor.log
sudo updatedb
locate servidor.log
locate -i SERVIDOR
```
**Qué decir:** "`locate` no busca en el disco, busca en una lista. Recién instalado, la lista no existe".
**Qué señalar:** el primer `locate` falla: `can not stat ()
'/var/lib/mlocate/mlocate.db': No such file or directory` — es esperado, no hay lista todavía. `sudo updatedb` la arma (tarda unos segundos:
está leyendo el disco entero). El segundo `locate` responde al instante
con la ruta completa. `-i` ignora mayúsculas.

El sistema corre `updatedb` solo una vez por día. Consecuencia: un
archivo creado hoy no aparece en `locate` hasta mañana (o hasta que
alguien corra `updatedb`). Por eso `find` es lo que se usa cuando se
busca algo reciente o con condiciones; `locate` cuando se busca un nombre
que existe hace tiempo.

Si `dnf` dice que `mlocate` no existe, es `plocate` (mismo uso).

---

## ¿Dónde está un programa? *(si hay tiempo)*

```bash
which tar
type tar
type ll
type cd
```
Ya lo vieron en el Bloque 1 (`type`). `which` solo busca programas en el
`$PATH`; `type` además reconoce alias y builtins. Un minuto y seguir.


touch ~/borrame.txt
sudo updatedb
locate borrame.txt          # aparece
rm ~/borrame.txt
locate borrame.txt          # sigue apareciendo, aunque ya no existe
sudo updatedb
locate borrame.txt          # ahora no
