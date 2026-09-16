# 5 — Respaldos: `tar` y compresión (guía del instructor)

Cinco minutos: una vuelta completa (crear → listar → extraer → comparar)
sobre `~/empresa`, y listo. Los compresores se comparan en el lab.

**Qué es `tar`:** junta una carpeta entera (con todo lo de adentro) en un
solo archivo, conservando rutas, permisos y dueños. No comprime.
**Qué es `gzip`:** comprime un archivo para que ocupe menos. `tar` puede
llamarlo solo con la letra `z`.
**Qué es `.tar.gz`:** las dos cosas: la carpeta empaquetada y comprimida.
Es el formato de respaldo de Linux.
**Qué es `zip`:** empaqueta y comprime en un solo paso, pero no guarda
permisos ni dueños. Se usa para mandarle algo a alguien con Windows.

## Las letras de `tar`

**Qué decir:** "cuatro letras: `c` crear, `x` extraer, `t` listar, `f` el
archivo. `z` es 'con gzip'. La `f` va última porque le sigue el nombre".
**Qué señalar:** el error clásico es `tar -cfz x.tar.gz dir` — con la `f`en el medio, `tar` toma `z` como nombre del archivo y crea un archivo llamado `z`. **`f` siempre última.**

`-C carpeta` = "andá a esa carpeta primero". Sirve para no guardar rutas
largas: `-C ~ empresa` guarda `empresa/...` y no
`home/student/empresa/...`.

## Crear, listar, extraer, comprobar

```bash
cd /tmp
tar -czf empresa.tar.gz -C ~ empresa
ls -lh empresa.tar.gz
```
**Qué decir:** "crear, con gzip, en el archivo `empresa.tar.gz`; desde el home, la carpeta `empresa`".
**Qué señalar:** no muestra nada (sin `v`). El `.tar.gz` pesa unos KB;
`~/empresa` pesa más — se comprimió.

```bash
tar -tzf empresa.tar.gz | head -5
```
**Qué decir:** "`t` lista lo que hay adentro sin extraer nada".
**Qué señalar:** las rutas empiezan con `empresa/`, no con
`/home/student/` — por el `-C ~`. Es lo que después decide dónde caen
los archivos al extraer.

```bash
mkdir restaurar
tar -xzf empresa.tar.gz -C restaurar
ls restaurar/empresa
```
**Qué decir:** "`x` extrae; `-C restaurar` = ponelo ahí, no acá".
**Qué señalar:** sin `-C`, extraería en `/tmp` directo. Con `-C`, queda
`/tmp/restaurar/empresa/...`. Regla: **siempre restaurar en otra carpeta primero, mirar, y después mover lo que haga falta** — nunca extraer un respaldo encima del original.

```bash
diff -r ~/empresa restaurar/empresa && echo "IGUAL"
```
**Qué decir:** "`diff -r` compara dos carpetas enteras. Si no dice nada, son idénticas".
**Qué señalar:** sale solo `IGUAL`. Eso es un respaldo verificado. Si
`diff` mostrara algo, habría una diferencia y el `echo` no correría (el
`&&` es "solo si lo anterior salió bien" — `$?` = 0).

## Compresores

No demostrar; la tabla queda y se comparan en el lab. Una frase: "`gzip`
es el estándar; `xz` comprime más pero tarda mucho más. Para respaldos diarios, `gzip`".

```bash
rm -r empresa.tar.gz restaurar
cd
```
Limpieza. Cerrar con: "en el lab van a respaldar `~/empresa` con fecha
en el nombre, restaurarlo, y después respaldar `/etc` — que es lo que se
hace en la vida real antes de tocar cualquier configuración".

## Anatomía del comando

```
tar  -czf  empresa.tar.gz  -C ~  empresa
      │││       │           │      └ qué meter adentro: la carpeta empresa
      │││       │           └ antes de empezar, pararse en ~ (el home)
      │││       └ el nombre del paquete que se va a crear
      ││└ f = "el paquete se llama..." → la palabra que sigue es el nombre
      │└ z = comprimir con gzip
      └ c = crear
```
