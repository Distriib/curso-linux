# Comandos — procesos

## Un programa corriendo es un proceso

```bash
ps
pstree -p | head -8
```

- **PID** = número del proceso. **PPID** = número de su padre.
- Todo nace de **PID 1**, `systemd`. La cadena de tu terminal: `systemd → sshd → sshd → bash`.
- Lo que va entre corchetes (`[kworker/0:1]`) son hilos del kernel: hijos de PID 2, no tienen programa en disco.

## Ver los procesos

| Comando | Qué muestra |
|---|---|
| `ps` | solo los de esta terminal |
| `ps aux` | todos, con `%CPU`, `%MEM` y estado |
| `ps -ef` | todos, con el PPID |
| `ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu \| head -8` | los 8 que más CPU usan (el `-` es "de mayor a menor") |
| `ps -C yes -o pid,ni,%cpu,cmd` | solo los que se llaman `yes`, columnas a elección |
| `pstree -p` | el árbol completo con PIDs |
| `pstree -ps 2351` | los ancestros del PID 2351 |
| `pgrep -a sshd` | PID y comando de los que se llaman `sshd` |
| `top` | monitor en vivo |
| `top -b -n 1 \| head -12` | una foto de `top` para pegar en un ticket |

## Una línea de `ps`

```
    PID    PPID USER     STAT CMD
   2351    2350 student  Ss   -bash
    │       │     │       │    └ el comando
    │       │     │       └ estado
    │       │     └ quién lo lanzó
    │       └ su padre
    └ su número
```

| `STAT` | Significa |
|---|---|
| `R` | corriendo o listo para correr |
| `S` | dormido, esperando algo (teclado, red, reloj). El 95 % está así |
| `D` | esperando al disco. **No se puede matar** hasta que el disco responda |
| `T` | detenido (Ctrl+Z, o señal `STOP`) |
| `Z` | zombie: terminó, pero el padre no recogió el resultado |
| `s` `+` `N` `<` | sufijos: líder de sesión · en primer plano · prioridad baja · prioridad alta |

## Teclas de `top`

| Tecla | Qué hace |
|---|---|
| `P` | ordenar por CPU (así arranca) |
| `M` | ordenar por memoria |
| `1` | una línea por núcleo |
| `u` | filtrar por usuario (escribir el nombre, Enter) |
| `c` | comando completo |
| `k` | matar: pide el PID, después la señal (Enter = 15) |
| `q` | salir |

Cabecera de `top`: primera línea = `uptime`; `Tasks` = cuántos en cada estado; `MiB Mem` = `free`.

## Señales — cómo hablarle a un proceso

| Nº | Nombre | Qué hace |
|---:|---|---|
| 1 | `HUP` | "colgar": los servicios releen su configuración; es lo que recibe un proceso al cerrar la terminal |
| 2 | `INT` | Ctrl+C |
| 15 | `TERM` | terminar de forma ordenada. **El default de `kill`.** Siempre primero |
| 9 | `KILL` | el kernel lo elimina sin avisarle. Solo si `TERM` no funcionó. No sirve en estado `D` |
| 19 / 18 | `STOP` / `CONT` | congelar / seguir |
| 20 | `TSTP` | Ctrl+Z |

| Comando | Qué hace |
|---|---|
| `kill 3001` | manda `TERM` al PID 3001 |
| `kill -9 3001` | manda `KILL` |
| `kill -STOP 3001` / `kill -CONT 3001` | pausa / reanuda |
| `kill -l` | lista las señales |
| `pgrep -a yes` | **mirar antes**: qué se va a matar |
| `pkill yes` | matar todos los que se llaman `yes` |
| `killall yes` | igual, nombre exacto |

## Jobs — trabajos de esta terminal

| Comando | Qué hace |
|---|---|
| `comando &` | lanzarlo en segundo plano; devuelve `[1] PID` |
| Ctrl+Z | suspender el que está en primer plano (queda `T`) |
| `jobs` | listar los trabajos de esta terminal |
| `bg %1` | reanudar el trabajo 1 en segundo plano |
| `fg %1` | traer el trabajo 1 al frente |
| `kill %1` | matar el trabajo 1 |
| `nohup comando &` | que sobreviva al cierre de la terminal |

## Prioridad — `nice`

De `-20` (máxima) a `19` (mínima); `0` es lo normal. Solo root puede **bajar** el número.

```bash
nice -n 10 comando
renice -n 15 -p 3102
sudo renice -n -5 -p 3102
```

Solo se nota cuando dos procesos **compiten** por el mismo núcleo. `taskset -c 0 comando` lo obliga a usar la CPU 0.

## Carga y memoria

```bash
nproc
uptime
free -m
vmstat 1 5
```

- **Load average**: promedio de procesos en `R` o `D` a 1, 5 y 15 minutos. Se lee contra `nproc`: con 2 núcleos, `2.00` es 100 %.
- En `free -m` importa `available`: lo que puede usar un programa nuevo. `buff/cache` es memoria prestada al disco; se devuelve.
- `vmstat 1 5`: cinco muestras, una por segundo. `r` = cola de CPU, `b` = esperando disco, `si`/`so` = swap (tienen que ser 0), `wa` = CPU esperando al disco.

## `/proc` — la ventana al kernel

```bash
head -8 /proc/1/status
sudo ls -l /proc/1/exe
```

`/proc/PID/status` = estado y dueño · `/proc/PID/exe` = el programa **real** aunque se llame distinto · `/proc/PID/cwd` = en qué carpeta está parado. `ps` y `top` leen de acá.
