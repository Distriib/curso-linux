# Lab — Proceso descontrolado (comandos)

Guiado, foto por parte. Todo como `student`. Es el lab donde más se desvía el tiempo (se quedan jugando con `top`): marcar los 25 minutos.

## Parte 1 — Saturar la CPU
```bash
yes > /dev/null &
yes > /dev/null &
jobs
sleep 20
uptime
```
Dejarlos corriendo.

## Parte 2 — Encontrarlos y matar uno desde `top`
```bash
top
```
`1` → las dos CPU al 100 %. `P` → los dos `yes` arriba. `k` → PID del primer `yes` → Enter → Enter (acepta `15/sigterm`). Ver que desaparece. `q`.
Qué decir: "Enter en la señal manda `TERM`. Si escribieran 9, sería `KILL`."

## Parte 3 — Matar el otro por nombre
```bash
pgrep -a yes
pkill yes
pgrep -a yes
jobs
```
Si a alguien le sigue vivo: `pkill -9 yes`. Y decir que fue la excepción, no la regla.

## Parte 4 — Congelar y descongelar
```bash
kill -l | head -3
sleep 600 &
kill -STOP %1
ps -o pid,stat,cmd -C sleep
kill -CONT %1
ps -o pid,stat,cmd -C sleep
kill %1
```
Después del `kill -STOP`, la shell puede imprimir `[1]+ Stopped sleep 600` al siguiente Enter: es normal.

## Parte 5 — Ctrl+Z, `bg`, `fg`
```bash
sleep 300
```
**Ctrl+Z**.
```bash
jobs
bg %1
jobs
fg %1
```
**Ctrl+C**.
Si alguien cierra la terminal con trabajos suspendidos, bash avisa `There are stopped jobs`: `jobs`, `fg`, Ctrl+C.

## Parte 6 — Sobrevivir al cierre de la terminal
```bash
nohup sleep 900 > /dev/null 2>&1 &
exit
```
Cada uno vuelve a entrar con `ssh -p 2222 student@localhost`.
```bash
pgrep -a sleep
ps -o pid,ppid,cmd -C sleep
pkill sleep
```
Qué decir: "el PPID ahora es 1: al morir su padre, lo adoptó systemd. Esto sirve para una emergencia; para algo permanente, en el bloque que sigue hacemos un servicio."

## Parte 7 — Prioridad
```bash
taskset -c 0 yes > /dev/null &
taskset -c 0 nice -n 10 yes > /dev/null &
sleep 5
ps -C yes -o pid,ni,%cpu,cmd
```
Leer el PID del `yes` con `NI 10` (acá `3102`). Cada uno con su número:
```bash
renice -n 19 -p 3102
renice -n 0 -p 3102
sudo renice -n -5 -p 3102
top -b -n 2 -d 5 | grep yes
pkill yes
```
El `renice -n 0` **tiene que fallar** con `Permission denied`: un usuario no puede mejorar la prioridad, ni volver a 0. Se mide con `top -b` y no con `ps` porque el `%CPU` de `ps` es el promedio de toda la vida del proceso y tarda en reflejar el cambio; el de `top` es el del intervalo de 5 segundos. En la salida de `top`, mirar las dos últimas líneas (la segunda muestra): el de `NI -5` pasó a ~90.

## Parte 8 — Memoria y actividad
```bash
free -m
vmstat 1 5
```
Confirmar que no quedó nada: `jobs` vacío, `pgrep -a yes` sin salida, `uptime` con la carga bajando.
