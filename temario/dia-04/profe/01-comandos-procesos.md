# Comandos — procesos (guía del instructor)

Diez minutos, tipeando mientras explicás. Los comandos están en la pestaña de estudiantes; no leas las tablas enteras. Tres cosas que tienen que quedar: (1) todo proceso tiene un PID y un padre, y todo nace de PID 1; (2) `TERM` antes que `KILL`, y `pgrep -a` antes que `pkill`; (3) el load average se lee contra el número de núcleos. Lo demás lo practican en los dos labs.

**Qué es un proceso:** un programa en ejecución. El archivo `/usr/bin/bash` es la receta; cada vez que alguien lo ejecuta hay un plato distinto en la cocina, con su número de orden. Ese número es el **PID**.
**Qué es el PPID:** el PID del proceso padre, el que lo lanzó. Cuando tipeás `ls` en la terminal, `bash` es el padre de ese `ls`.
**Qué es el kernel:** el núcleo del sistema operativo. El programa que habla con el hardware y reparte CPU y memoria entre los procesos. Es lo que arranca primero.
**Qué es systemd / PID 1:** el primer proceso que lanza el kernel. Arranca todo lo demás. Por eso todos los procesos descienden de él. En el Bloque 2 se ve como administrador de servicios.
**Qué es un hilo:** una parte de un proceso que corre en paralelo con las otras, compartiendo memoria. En `pstree` van entre llaves `{}`.
**Qué son los hilos del kernel:** procesos que no son de nadie sino del propio kernel (`[kworker/0:1]`, `[kthreadd]`). Van entre corchetes, su padre es PID 2 y **no tienen un programa en disco**. Ese detalle es la clave del ticket 1 del reto: un impostor que se llame `kworkerd` sí tiene programa en disco.
**Qué es `sshd`:** el servicio de SSH. Hay tres en el árbol: el que escucha, uno por conexión como root, y uno por sesión como el usuario.
**Qué es un servicio (o demonio):** un programa que corre de fondo, sin terminal, esperando trabajo: `sshd`, `chronyd`. En RHEL los administra systemd (Bloque 2).
**Qué es `ps`:** *process status*. Lista procesos. Tiene dos sintaxis históricas: `ps aux` (sin guion, estilo BSD) y `ps -ef` (con guion, estilo Unix). Las dos muestran todo; la diferencia son las columnas.
**Qué es `ps -eo ...`:** `-e` = todos los procesos; `-o` = elegir las columnas, separadas por coma. Es la forma de armar exactamente la tabla que uno quiere.
**Qué es `--sort=-%cpu`:** ordenar por esa columna. El guion delante = de mayor a menor.
**Qué es `ps -C yes`:** solo los procesos cuyo comando se llama `yes`. **Qué es `ps -p 2351`:** solo el proceso con ese PID.
**Qué es `STAT`:** la columna de estado. `R` corriendo, `S` dormido esperando algo, `D` esperando al disco (no se puede matar), `T` detenido, `Z` zombie. Las letras chicas de atrás son detalles: `s` líder de sesión (el proceso que "manda" en esa terminal), `+` está en primer plano, `N` prioridad baja, `<` prioridad alta.
**Qué es un zombie:** un proceso que ya terminó pero cuyo padre todavía no recogió su resultado. No consume nada; es solo una entrada en la tabla. No se mata (ya está muerto): se arregla el padre.
**Qué es `pstree`:** dibuja el árbol de procesos. `-p` muestra los PIDs; `-s 2351` muestra solo los ancestros de ese PID hasta PID 1.
**Qué es `pgrep`:** busca procesos por nombre y devuelve sus PIDs. `-a` agrega la línea de comando completa. Es el mismo buscador que `pkill`, pero sin matar: por eso se usa **antes**.
**Qué es `top`:** monitor en vivo de procesos, ordenado por consumo, que se refresca cada 3 segundos. Se maneja con teclas de una letra. `top -b -n 1` es "modo batch": una sola foto, sin interactividad, para pegar en un ticket.
**Qué es `PR` y `NI` en top:** `NI` es el valor de `nice`; `PR` es la prioridad que calcula el kernel, `20 + NI`. **Qué es `RES`:** memoria real que ocupa. **Qué es `id` en la cabecera:** porcentaje de CPU ociosa (*idle*). **Qué es `us`:** CPU usada por programas de usuario.
**Qué es una señal:** un aviso numerado que se le manda a un proceso. El proceso puede reaccionar (releer configuración, cerrar ordenadamente) o, en el caso de `KILL`, no llega a reaccionar: el kernel lo elimina.
**Qué es `HUP`:** *hang up*, "colgar". Viene de los módems: se cortó la línea. Hoy los servicios la usan como "releé tu configuración", y es lo que recibe un programa cuando se cierra la terminal en la que corría.
**Qué es `TERM`:** "terminá ordenadamente". El proceso cierra archivos y conexiones y se va. Es lo que manda `kill` si no le decís otra cosa.
**Qué es `KILL`:** el kernel elimina el proceso sin avisarle. No puede cerrar nada: archivos a medio escribir, bases de datos corruptas. Solo si `TERM` no funcionó después de unos segundos. No funciona sobre un proceso en estado `D`.
**Qué es `STOP` / `CONT`:** congelar y descongelar. El proceso queda en `T` sin consumir CPU, y sigue exactamente donde estaba.
**Qué es `kill`:** manda una señal a un PID. `kill 3001` manda `TERM`; `kill -9 3001` manda `KILL`; `kill -STOP 3001`. `kill -l` lista las 64 señales con su número.
**Qué es `pkill`:** `kill` por nombre: mata **todos** los procesos que se llaman así. **Qué es `killall`:** igual, con nombre exacto.
**Qué es un job:** un trabajo lanzado desde esta terminal. `comando &` lo manda a segundo plano y la shell responde `[1] 3001`: trabajo número 1, PID 3001. `%1` es "el trabajo 1" en `kill`, `bg`, `fg`.
**Qué es segundo plano / primer plano:** en primer plano el comando tiene el teclado y la terminal espera a que termine. En segundo plano corre mientras la terminal te devuelve el prompt.
**Qué es Ctrl+Z:** manda `TSTP` al programa en primer plano: lo suspende (estado `T`) y devuelve el prompt. **No lo cierra.** `jobs` lo lista como `Stopped`; `bg %1` lo reanuda atrás; `fg %1` lo trae al frente.
**Qué es `nohup`:** *no hang up*. Lanza el comando inmune a `HUP`: cuando cerrás la terminal, no muere. Su salida va al archivo `nohup.out` si no la redirigís.
**Qué es `/dev/null`:** el agujero negro. Todo lo que se escribe ahí desaparece. `yes > /dev/null` = que `yes` escriba sin parar a la nada.
**Qué es `2>&1`:** los errores (salida 2) van al mismo lugar que la salida normal (salida 1). `> /dev/null 2>&1` = tirar todo, salida y errores.
**Qué es `yes`:** imprime la letra `y` infinitamente. Mandado a `/dev/null` es la forma más simple de quemar una CPU a propósito.
**Qué es `sleep 600`:** no hace nada durante 600 segundos. Sirve como proceso inofensivo para practicar señales.
**Qué es `nice`:** la prioridad de un proceso, de `-20` (la máxima) a `19` (la mínima). `0` es lo normal. Un número **más alto** es **menos** prioridad: el proceso es "más amable" con los demás. `nice -n 10 comando` lo lanza con 10; `renice -n 15 -p 3102` se lo cambia a uno que ya corre (`-p` = por PID). Un usuario normal solo puede subir el número (empeorar); bajarlo es de root.
**Qué es `taskset -c 0`:** obliga al comando a usar solo la CPU número 0. En la VM hay 2 núcleos: sin esto, dos `yes` se reparten uno por núcleo y el `nice` no se nota, porque no compiten.
**Qué es un núcleo:** una unidad de la CPU que ejecuta un proceso a la vez. `nproc` dice cuántos hay: la VM tiene 2.
**Qué es `uptime`:** hace cuánto arrancó la máquina y el **load average**.
**Qué es el load average:** el promedio de procesos que están corriendo o esperando disco (`R` o `D`), a 1, 5 y 15 minutos. Se lee contra `nproc`: con 2 núcleos, `1.00` es 50 % y `2.00` es 100 %. `4.00` es "hay el doble de trabajo del que cabe".
**Qué es `free -m`:** memoria en megabytes. Lo que importa es la columna `available`: lo que puede usar un programa nuevo. `buff/cache` es memoria que el kernel presta para acelerar el disco y devuelve cuando hace falta; por eso `free` (libre) parece poco y no es problema.
**Qué es la swap:** espacio en disco que se usa como memoria cuando la RAM se acaba. Es lentísima. `Swap: used 0` es lo deseable.
**Qué es `vmstat 1 5`:** cinco muestras de actividad, una por segundo. `r` = procesos esperando CPU; `b` = esperando disco; `si`/`so` = memoria yendo y viniendo de la swap (tienen que ser 0); `us`/`sy`/`id`/`wa` = CPU en programas / en kernel / ociosa / esperando al disco. La primera fila es el promedio desde el arranque, no una muestra real.
**Qué es `/proc`:** una carpeta que no está en el disco: es una ventana al kernel. `/proc/2351/` tiene todo sobre el proceso 2351. `ps` y `top` no saben nada por sí mismos: leen de acá.
**Qué es `/proc/PID/status`:** nombre, estado, dueño, memoria del proceso. **Qué es `/proc/PID/exe`:** un enlace al programa **real** que está corriendo, aunque el proceso se haya renombrado. **Qué es `/proc/PID/cwd`:** la carpeta en la que está parado.

---

## Un programa corriendo es un proceso

`ps` y `pstree -p | head -8`.
**Qué decir:** "un programa en disco es una receta; un proceso es el plato en la cocina. Cada plato tiene un número: el PID."
**Qué señalar:** en `ps`, dos líneas: `bash` (tu shell) y `ps` (el que acabás de correr, hijo de bash). En `pstree`, `systemd(1)` arriba de todo y todo cuelga de él.

---

## Ver los procesos

No demostrar la tabla: se hace en el Lab 1. Una frase: "`ps aux` y `ps -ef` son las dos formas viejas; la que uso yo cuando hay un problema es la de `--sort=-%cpu`: los que más CPU usan, primero."

---

## Una línea de `ps` y `STAT`

**Qué señalar:** de la tabla de estados, dos filas: `S` (dormido, casi todos) y `D` (esperando disco). **La frase exacta:** *"un proceso en `D` no se puede matar ni con `-9`. Si ven muchos `D`, el problema no es el proceso: es el disco o la red."*

---

## Teclas de `top`

No leerlas: se practican en el Lab 1, Parte 5. Decir solo: "`P` por CPU, `M` por memoria, `k` para matar, `q` para salir. Ctrl+C también sale, y no mata nada."

---

## Señales

**Qué decir, despacio:** *"`kill` no significa matar: significa mandar una señal. La que manda por defecto, `TERM`, es 'terminá ordenadamente'. `-9` es la última opción, porque el proceso no llega a cerrar nada."*
**Qué señalar:** la fila `pgrep -a yes` antes de `pkill yes`. "Primero miro qué voy a matar. Después mato."

---

## Jobs

Una frase: "el `&` manda el comando atrás; Ctrl+Z lo congela; `jobs` te dice qué tenés colgado. Error clásico: creer que Ctrl+Z cerró el `vim` y dejar diez `vim` suspendidos." Lo practican en el Lab 2.

---

## Prioridad

**Qué decir:** "`nice` va de -20 a 19 y cuanto más alto, menos prioridad. Solo root puede bajar el número." Y anticipar la trampa del Lab 2, Parte 7: *"el `nice` solo se nota si dos procesos pelean por el mismo núcleo. Con 2 núcleos y 2 procesos, cada uno tiene el suyo y no pelean. Por eso vamos a usar `taskset` para meterlos en el mismo."*

---

## Carga y memoria

`nproc`, `uptime`, `free -m`.
**Qué señalar:** `nproc` da `2`. En `uptime`, los tres números del load average, ahora cerca de 0. **La frase exacta:** *"el load average se divide por el número de núcleos. `2.00` acá es 100 %; en un servidor de 16 núcleos sería nada."* En `free -m`, señalar la última columna, `available`: "esa es la memoria de verdad libre; ignoren `free`."

---

## `/proc`

`head -8 /proc/1/status` y `sudo ls -l /proc/1/exe`.
**Qué señalar:** `Name: systemd`, `Pid: 1`, `PPid: 0` (no tiene padre). El `exe` apunta a `/usr/lib/systemd/systemd`. **Qué decir:** "cuando un proceso se llama `kworkerd` pero su `exe` apunta a `/tmp/algo`, alguien te está mintiendo. Guarden esa idea para el reto."
