# Lab 2 — Proceso descontrolado

Vamos a provocar dos procesos que se comen la CPU, encontrarlos y terminarlos; después pausar, reanudar, sobrevivir al cierre de la terminal y bajar la prioridad. Todo como `student`.

---

## Parte 1 — Saturar la CPU

**¿Cómo se ve en `uptime` un servidor sobrecargado?**

```bash
yes > /dev/null &
yes > /dev/null &
jobs
```

Esperar 20 segundos:

```bash
sleep 20
uptime
```

Foto.

**Comprobar:**
```
[1] 3001
[2] 3002
[1]-  Running                 yes > /dev/null &
[2]+  Running                 yes > /dev/null &
 09:45:33 up 43 min,  1 user,  load average: 1.42, 0.45, 0.17
```
`yes` imprime "y" sin parar; mandado a `/dev/null`, solo quema CPU. El primer número del load average sube hacia `2.00`: dos procesos en `R` sobre dos núcleos. **Dejarlos correr.**

---

## Parte 2 — Encontrarlos y matar uno desde `top`

```bash
top
```

1. `1`: las dos CPU cerca de `100.0 us`.
2. Los dos `yes` arriba, con `~99` en `%CPU` y estado `R`.
3. `k`, escribir el PID del **primer** `yes`, Enter, y en `Send pid ... signal [15/sigterm]` Enter otra vez.
4. Ver que desaparece. `q`.

Foto antes del `q`.

**Comprobar:** queda **un** `yes` en la lista. Al salir, la shell avisa `[1]-  Terminated   yes > /dev/null`.

---

## Parte 3 — Matar el otro por nombre

**¿Qué se hace siempre antes de `pkill`?**

```bash
pgrep -a yes
pkill yes
pgrep -a yes
jobs
```

Foto.

**Comprobar:**
```
3002 yes
[2]+  Terminated              yes > /dev/null
```
El segundo `pgrep` no imprime nada: no queda ninguno. `pkill` manda `TERM` (15). `-9` solo si `TERM` no alcanzó.

---

## Parte 4 — Congelar y descongelar

**¿Cómo pauso un respaldo pesado en horario de oficina sin matarlo?**

```bash
kill -l | head -3
sleep 600 &
kill -STOP %1
ps -o pid,stat,cmd -C sleep
kill -CONT %1
ps -o pid,stat,cmd -C sleep
kill %1
```

Foto.

**Comprobar:**
```
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
[1] 3050
    PID STAT CMD
   3050 T    sleep 600
    PID STAT CMD
   3050 S    sleep 600
[1]+  Terminated              sleep 600
```
`STOP` lo deja en `T`; `CONT` lo devuelve a `S`. `%1` es el trabajo 1 de esta terminal.

---

## Parte 5 — Ctrl+Z, `bg`, `fg`

**¿Ctrl+Z cierra el programa?**

```bash
sleep 300
```

Pulsar **Ctrl+Z**. Después:

```bash
jobs
bg %1
jobs
fg %1
```

Pulsar **Ctrl+C**. Foto.

**Comprobar:**
```
[1]+  Stopped                 sleep 300
[1]+  Stopped                 sleep 300
[1]+ sleep 300 &
[1]+  Running                 sleep 300 &
sleep 300
```
Ctrl+Z **no** mata: suspende. `bg` lo reanuda atrás, `fg` lo trae al frente, y ahí Ctrl+C sí lo termina.

---

## Parte 6 — Sobrevivir al cierre de la terminal

**¿Qué pasa con lo que dejo corriendo si cierro la sesión SSH?**

```bash
nohup sleep 900 > /dev/null 2>&1 &
exit
```

Volver a entrar (`ssh -p 2222 student@localhost`) y:

```bash
pgrep -a sleep
ps -o pid,ppid,cmd -C sleep
pkill sleep
```

Foto.

**Comprobar:**
```
3080 sleep 900
    PID    PPID CMD
   3080       1 sleep 900
```
Sobrevivió al `exit` y ahora su padre es PID 1. Sin `nohup`, la shell le habría mandado `HUP` al cerrar. Para algo permanente, lo correcto es un servicio (Bloque 2).

---

## Parte 7 — Prioridad: dos procesos peleando por un núcleo

**¿Cómo hago que un trabajo pesado no moleste a los demás?**

```bash
taskset -c 0 yes > /dev/null &
taskset -c 0 nice -n 10 yes > /dev/null &
sleep 5
ps -C yes -o pid,ni,%cpu,cmd
```

Anotar el PID del `yes` con `NI 10` (acá `3102`). Con **ese número**:

```bash
renice -n 19 -p 3102
renice -n 0 -p 3102
sudo renice -n -5 -p 3102
top -b -n 2 -d 5 | grep yes
pkill yes
```

Foto.

**Comprobar:**
```
    PID  NI %CPU CMD
   3101   0 90.5 yes
   3102  10  9.4 yes
3102 (process ID) old priority 10, new priority 19
renice: failed to set priority for 3102 (process ID): Permission denied
3102 (process ID) old priority 19, new priority -5
   3101 student   20   0 ...  R   9.3   0.0 ...  yes
   3102 student   15  -5 ...  R  90.1   0.0 ...  yes
```
Con `nice 10` recibe ~10 % contra ~90 %. Un usuario puede **empeorar** su prioridad (19) pero no mejorarla, ni volver a 0. Root sí (`-5`), y la proporción se invierte. En `top`, `PR = 20 + NI`.

---

## Parte 8 — Memoria y actividad

```bash
free -m
vmstat 1 5
```

Foto.

**Comprobar:**
```
Mem:            3713         352        3010           8         350        3129
Swap:           4027           0        4027
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 3082468   2104 356420    0    0    12     3   80  150  0  0 99  0  0
 0  0      0 3082468   2104 356420    0    0     0     0   62  110  0  0 100 0  0
...
```
En `free -m`, la última columna (`available`, 3129) es lo que puede usar un programa nuevo. Swap en 0: bien. En `vmstat`, la primera fila es el promedio desde el arranque; las siguientes son reales. `si`/`so` distintos de 0 = falta memoria; `wa` alto = el disco es el cuello de botella.
