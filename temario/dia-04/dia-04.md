# Día 04 — Procesos, systemd, logs y gestión de software
> Al terminar el día, el participante identifica y controla lo que corre en el servidor (procesos, prioridades, señales), administra servicios con `systemd` y crea una unidad propia, lee e interpreta los logs con `journalctl` y `rsyslog`, e instala, consulta y deshace software con `rpm` y `dnf`, incluidos repositorios externos (EPEL) y un repositorio local desde la ISO para servidores sin internet.

**Ficha técnica cubierta:**
- RH124 M4 Administración del Sistema: Procesos, Servicios, Logs del sistema (módulo completo).
- RH124 M6 Gestión de Software: RPM, DNF/YUM, Repositorios (módulo completo).
- Adelanta de RH134 M8 (Logs avanzados) el journal persistente y `journalctl` con filtros; de RH134 M2 la noción de unidades `timer` (`logrotate.timer`, `dnf-makecache.timer`).

**Requisitos previos:**
- Snapshot `dia03-fin` tomado; VM `rhel01` encendida y accesible con `ssh -p 2222 student@localhost`.
- VM registrada: `sudo subscription-manager status` responde `Overall Status: Registered` (o `Disabled` con la línea "Content Access Mode is set to Simple Content Access", que es normal) y `sudo dnf repolist` lista `rhel-9-for-<arch>-baseos-rpms` y `rhel-9-for-<arch>-appstream-rpms`. Hoy se instala software: sin registro, nada del Bloque 4 funciona.
- Usuarios del Día 03 (`ana`, `carlos`, `pedro`, `laura`): el reto usa `pedro`. Si no existe, el script de rotura lo crea.
- Paquetes que ya vienen en la instalación "Server": `procps-ng` (ps, top, free, vmstat, pgrep), `psmisc` (pstree, killall), `util-linux` (renice, taskset, setsid), `rsyslog`, `chrony`, `logrotate`, `dnf-plugins-core`, `tuned`. Comprobar al inicio con `rpm -q procps-ng psmisc util-linux rsyslog chrony logrotate dnf-plugins-core tuned`; si falta alguno, `sudo dnf install -y psmisc dnf-plugins-core tuned`.
- `tree` instalado (viene del Día 02): el Lab 4.1 lo quita y lo vuelve a instalar desde un `.rpm`. Comprobar con `rpm -q tree`; si no está, `sudo dnf install -y tree` antes de empezar.
- **ISO DVD de RHEL 9** (la misma del Día 01, ~10 GB) a mano en el equipo propio: en el Lab 4.3 se conecta a la VM como CD virtual **sin apagarla**. La "Boot ISO" pequeña no sirve (no trae paquetes).
- Conexión a internet en la VM (`ping -c 2 dl.fedoraproject.org`) para EPEL y CodeReady Builder.

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:05 | 5 | Repaso | Tres preguntas del Día 03 con la terminal abierta; objetivos del día |
| 0:05–0:20 | 15 | Bloque 1 — Conceptos | Procesos: PID/PPID, árbol, estados, señales, jobs, prioridades, carga y memoria, `/proc` |
| 0:20–0:35 | 15 | Lab 1.1 | Radiografía de procesos: `ps`, `pstree`, `top`, `/proc/PID` |
| 0:35–0:55 | 20 | Lab 1.2 | Proceso runaway: encontrarlo y matarlo; señales, jobs, `nohup`, `nice`/`renice`, `uptime`, `free`, `vmstat` |
| 0:55–1:10 | 15 | Bloque 2 — Conceptos | systemd: tipos de unidad, `systemctl`, activo vs habilitado, mask, targets, unit files |
| 1:10–1:30 | 20 | Lab 2.1 | Controlar servicios con `sshd` y `httpd`: status/start/enable/mask, `cat`/`show`, dependencias, targets, timers y sockets |
| 1:30–1:45 | 15 | Lab 2.2 | Unidad propia `monitor.service` con `Restart=on-failure`; hora del sistema con `chronyd` y `timedatectl` |
| 1:45–2:00 | 15 | Descanso | |
| 2:00–2:10 | 10 | Bloque 3 — Conceptos | Dos sistemas de log: `rsyslog` (`/var/log`) y `journald`; facilidad.prioridad; `logrotate` |
| 2:10–2:22 | 12 | Lab 3.1 | `logger`, regla propia en `/etc/rsyslog.d/`, intentos de acceso fallidos, `journalctl` con filtros |
| 2:22–2:30 | 8 | Lab 3.2 | Journal persistente y `logrotate` para `/var/log/monitor.log` |
| 2:30–2:45 | 15 | Bloque 4 — Conceptos | RPM (nombre-versión-release.arch, base de datos, firmas GPG), DNF (repos, transacciones, módulos), repositorios |
| 2:45–3:05 | 20 | Lab 4.1 | Consultar con `rpm -q*`, instalar/quitar con `dnf`, `rpm -ivh` de un archivo, `history undo`, `provides`, módulos |
| 3:05–3:15 | 10 | Lab 4.2 | Repositorios Red Hat, CodeReady Builder y EPEL: instalar `htop`; habilitar/deshabilitar repos |
| 3:15–3:25 | 10 | Lab 4.3 | Repositorio local desde la ISO DVD (servidor sin internet) |
| 3:25–3:50 | 25 | Reto individual | Tres tickets: proceso disfrazado, servicio enmascarado, paquete de origen desconocido |
| 3:50–4:00 | 10 | Cierre | Resumen, cheatsheet, snapshot `dia04-fin`, tarea |

## Prioridad si falta tiempo

**Imprescindible**
- Encontrar el proceso que consume CPU con `top` (tecla `P`) y con `ps -eo ... --sort=-%cpu`, y terminarlo con `kill` (15 antes que 9) o `pkill`.
- Diferencia entre **activo** (`start`/`stop`, `is-active`) y **habilitado** (`enable`/`disable`, `is-enabled`); `enable --now`; leer `systemctl status` completo.
- Crear `monitor.service` en `/etc/systemd/system/`, `daemon-reload`, `enable --now`, verlo en `journalctl -u monitor`.
- `journalctl -u`, `-f`, `-b`, `-p err`, `--since`; `/var/log/messages` y `/var/log/secure`; `logger` para probar.
- `dnf search/info/install/remove`, `dnf history` + `history undo`, `rpm -qf`, `rpm -ql`, `rpm -qi`.
- `dnf repolist`; qué pasa sin registro; instalar EPEL y saber qué riesgos tiene.

**Importante**
- Señales (`HUP`, `TERM`, `KILL`, `STOP`/`CONT`), jobs (`&`, Ctrl+Z, `bg`, `fg`, `nohup`), `nice`/`renice`, load average vs núcleos, `free`, `vmstat`.
- `mask`/`unmask` y por qué existe; targets (`get-default`/`set-default`); `Restart=on-failure` en acción.
- `chronyc sources`, `timedatectl` y por qué la hora importa para los logs.
- Journal persistente; `journalctl --disk-usage`, `-b -1`, `-o verbose`.
- Regla `facilidad.prioridad` en `/etc/rsyslog.d/`; `logrotate -d`.
- CodeReady Builder; `dnf config-manager --set-disabled`; firmas GPG (`rpm -K`, importación de llaves).
- Repositorio local desde la ISO (Lab 4.3): es el caso real de la institución; si no da tiempo, demo del instructor y tarea.

**Si sobra tiempo**
- Zombies y estado `D`; `taskset`; `tuned-adm`.
- `systemctl edit` (drop-ins), `Type=oneshot`, `systemd-analyze blame`, `cockpit.socket` como ejemplo de activación por socket.
- `dnf module install nodejs:20`, `dnf group install "Headless Management"`, `dnf needs-restarting`, `dnf update --security`.
- Lab 4.1 pasos 7 y 8 (grupos, módulos, `clean all`/`makecache`): si el bloque va justo, el instructor los muestra en su pantalla y los participantes solo miran.
- `createrepo_c` para un repositorio de RPMs propios.

## Repaso del Día 03 (5 min)

Tres preguntas rápidas; un participante distinto responde cada una ejecutando el comando:

1. ¿Cómo agrego a `carlos` al grupo `auditoria` sin sacarlo de sus otros grupos? (`sudo usermod -aG auditoria carlos`; el `-a` es la clave.)
2. ¿Qué permisos pongo a `/srv/sistemas` para que los archivos nuevos hereden el grupo? (`sudo chmod 2770 /srv/sistemas`; el `2` es setgid.)
3. ¿Cómo veo qué puede ejecutar `pedro` con `sudo` sin ser `pedro`? (`sudo -l -U pedro`.)

Cierre del repaso: ayer controlamos **quién** puede hacer qué. Hoy vemos **qué está pasando** en el servidor (procesos y servicios), **cómo se registra** (logs) y **de dónde sale el software** que corre ahí.

## Bloque 1 — Procesos

### Conceptos (15 min)

**Qué es un proceso.** Un programa en ejecución: código cargado en memoria, con un identificador único (**PID**), un padre (**PPID**), un usuario dueño, un estado, una prioridad y unos descriptores de archivo abiertos. Analogía: el binario `/usr/bin/vim` es una receta; cada vez que alguien la ejecuta hay un plato distinto en la cocina, con su número de orden (PID) y el mesero que lo pidió (PPID).

**Todo nace de PID 1.** El kernel arranca un solo proceso, `systemd` (PID 1), y todo lo demás es descendiente suyo: `systemd` → `sshd` → `sshd` de la sesión → `bash` → `ps`. `pstree` lo dibuja. Si un padre muere antes que sus hijos, los huérfanos son adoptados por PID 1 (o por el `systemd --user` de la sesión). Los hilos del kernel (`[kworker/0:1]`, `[kthreadd]`) aparecen entre corchetes, tienen PPID 2 y no tienen binario en disco: ese detalle sirve para desenmascarar impostores (Reto, ticket 1).

**Estados** (columna `STAT` de `ps`, `S` de `top`):
- `R` running o listo para correr.
- `S` sleeping interrumpible: espera un evento (teclado, red, temporizador). El 95 % de los procesos de un servidor están así.
- `D` sleeping **no** interrumpible: espera E/S de disco o red. **No se puede matar** ni con `-9` hasta que la E/S termine; muchos `D` indican un disco o un NFS con problemas y elevan el load average aunque la CPU esté libre.
- `T` detenido (Ctrl+Z, `SIGSTOP`).
- `Z` zombie: terminó pero su padre no ha recogido el código de salida (`wait()`). No consume CPU ni memoria, solo una entrada en la tabla; se elimina al morir el padre.
- Sufijos: `s` líder de sesión, `+` en primer plano, `<` prioridad alta, `N` prioridad baja (`nice`), `l` multihilo.

**Señales.** La forma de hablarle a un proceso. `kill -l` lista las 64. Las que importan:

| Nº | Nombre | Qué hace | Cuándo usarla |
|---:|---|---|---|
| 1 | `HUP` | "Colgar": los demonios la usan para releer su configuración | `kill -HUP` a sshd/rsyslog; también es lo que recibe un proceso al cerrar la terminal |
| 2 | `INT` | Interrupción, es Ctrl+C | Primer plano |
| 15 | `TERM` | Terminar de forma ordenada; **el default de `kill`** | Siempre primero: el proceso cierra archivos y conexiones |
| 9 | `KILL` | El kernel lo elimina sin avisarle | Solo si `TERM` no funcionó tras unos segundos; no permite limpieza; no funciona en estado `D` |
| 19 / 18 | `STOP` / `CONT` | Congelar / reanudar | Pausar un proceso pesado sin matarlo |
| 20 | `TSTP` | Ctrl+Z (stop desde la terminal) | Jobs |

Herramientas: `kill PID` (por número), `pkill patrón` (por nombre, expresión regular; `-u usuario`, `-f` busca en toda la línea de comando), `killall nombre` (nombre exacto). `pgrep` es el mismo motor de búsqueda sin matar: **siempre `pgrep -a` antes de `pkill`** para ver qué se va a matar.

**Jobs.** Control de trabajos de la shell: `comando &` lo lanza en segundo plano; Ctrl+Z lo suspende; `bg` lo reanuda en segundo plano; `fg` lo trae al frente; `jobs` los lista; `kill %1` mata el job 1. Al cerrar la sesión SSH, la shell manda `SIGHUP` a sus jobs: `nohup comando &` lo inmuniza (y guarda la salida en `nohup.out`). Para trabajos largos en producción se prefiere `tmux` o, mejor, un servicio de systemd (Bloque 2).

**Prioridad (`nice`).** Valor de -20 (máxima prioridad) a 19 (mínima); 0 por defecto. Solo root puede **bajar** el número (subir la prioridad). `nice -n 10 comando` lanza con prioridad reducida; `renice -n 15 -p PID` la cambia en caliente. En `top`, `PR = 20 + NI`. Importante: el `nice` solo se nota cuando hay **competencia** por la CPU; en una VM de 2 núcleos con 2 procesos, ambos corren al 100 % sin importar el `nice`.

**Carga y memoria.** `uptime` muestra el *load average* a 1, 5 y 15 minutos: el promedio de procesos en `R` o `D`. Se lee **relativo al número de núcleos** (`nproc`): en nuestra VM de 2 vCPU, 1.00 es 50 % de ocupación, 2.00 es 100 % y 4.00 es "hay el doble de trabajo del que cabe". `free -m` muestra memoria: lo que importa es `available`, no `free`; `buff/cache` es memoria que el kernel presta para acelerar disco y devuelve cuando hace falta. `vmstat 1 5` muestra cinco muestras con un segundo de intervalo: `r` (cola de CPU), `b` (bloqueados en E/S), `si`/`so` (swap: si no son 0 hay presión de memoria), `us`/`sy`/`id`/`wa` (CPU usuario/sistema/ocioso/esperando disco).

**`/proc`.** No es un directorio en disco, es una ventana al kernel. `/proc/PID/` tiene todo sobre un proceso: `status` (estado, UID, memoria), `cmdline` (línea de comando exacta), `exe` (enlace al binario real, aunque le hayan cambiado el nombre), `cwd` (directorio de trabajo), `fd/` (archivos abiertos), `environ`. `ps` y `top` solo leen de aquí.

### Lab 1.1 — Radiografía de procesos (15 min)

- **Objetivo:** leer el árbol de procesos de la VM con `ps`, `pstree`, `top` y `/proc`, e identificar qué corre, quién lo lanzó y en qué estado está.

1. Mi propio proceso y sus ancestros:

```bash
echo $$
ps -o pid,ppid,user,stat,cmd -p $$
pstree -ps $$
```
Salida esperada:
```text
2351
    PID    PPID USER     STAT CMD
   2351    2350 student  Ss   -bash
systemd(1)───sshd(890)───sshd(2345)───sshd(2350)───bash(2351)───pstree(2402)
```
Qué observar: `$$` es el PID de la shell actual. La cadena `systemd → sshd → sshd → sshd → bash`: el primer `sshd` es el demonio, el segundo la conexión (como root, separación de privilegios), el tercero la sesión ya como `student`. `Ss`: dormida (`S`) y líder de sesión (`s`).

2. Todos los procesos, en los dos formatos clásicos:

```bash
ps aux | head -5
ps -ef | head -5
ps aux | wc -l
```
Salida esperada:
```text
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.4 107324 17024 ?        Ss   09:02   0:02 /usr/lib/systemd/systemd --switched-root --system --deserialize 31
root           2  0.0  0.0      0     0 ?        S    09:02   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   09:02   0:00 [rcu_gp]
...
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 09:02 ?        00:00:02 /usr/lib/systemd/systemd --switched-root --system --deserialize 31
root           2       0  0 09:02 ?        00:00:00 [kthreadd]
...
130
```
Qué observar: `aux` (sintaxis BSD, sin guion) muestra `%CPU`, `%MEM`, `STAT`; `-ef` (sintaxis Unix) muestra `PPID`. PID 2 `kthreadd` es el padre de todos los hilos del kernel, que van entre corchetes. Unos 120–150 procesos es lo normal en una instalación "Server".

3. El formato que se usa de verdad para diagnosticar: columnas a elección y orden por consumo:

```bash
ps -eo pid,ppid,user,%cpu,%mem,ni,stat,cmd --sort=-%cpu | head -8
ps -eo pid,user,%mem,rss,cmd --sort=-%mem | head -5
```
Salida esperada:
```text
    PID    PPID USER     %CPU %MEM  NI STAT CMD
      1       0 root      0.1  0.4   0 Ss   /usr/lib/systemd/systemd --switched-root --system --deserialize 31
    890       1 root      0.0  0.2   0 Ss   sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
    712       1 root      0.0  0.8   0 Ssl  /usr/lib/systemd/systemd-journald
...
```
Qué observar: `--sort=-%cpu` ordena descendente. Este comando (o `--sort=-%mem`) es lo primero que se ejecuta ante un ticket de "el servidor está lento".

4. El árbol completo y los procesos de un usuario:

```bash
pstree -p | head -15
ps -u root -o pid,cmd | wc -l
pgrep -a sshd
```
Salida esperada:
```text
systemd(1)─┬─NetworkManager(780)─┬─{NetworkManager}(782)
           │                     └─{NetworkManager}(784)
           ├─agetty(905)
           ├─auditd(690)───{auditd}(691)
           ├─chronyd(760)
           ├─crond(900)
           ├─dbus-broker-lau(720)───dbus-broker(722)
           ...
98
890 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
2345 sshd: student [priv]
2350 sshd: student@pts/0
```
Qué observar: las llaves `{}` son hilos del mismo proceso. `pgrep -a` muestra PID y línea de comando: es la forma segura de mirar antes de `pkill`.

5. `top` interactivo. Ejecutarlo y practicar las teclas en este orden; salir con `q`:

```bash
top
```
Salida esperada (cabecera):
```text
top - 09:40:12 up 38 min,  1 user,  load average: 0.05, 0.08, 0.04
Tasks: 130 total,   1 running, 129 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.2 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   3713.9 total,   3010.2 free,    353.4 used,    350.3 buff/cache
MiB Swap:   4027.0 total,   4027.0 free,      0.0 used.   3129.5 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2402 student   20   0   10952   4224   3456 R   0.3   0.1   0:00.02 top
      1 root      20   0  107324  17024  10880 S   0.0   0.4   0:02.11 systemd
```
Teclas: `P` ordena por CPU (default), `M` por memoria, `T` por tiempo acumulado, `1` muestra cada CPU por separado (verán 2 líneas `%Cpu0`/`%Cpu1`), `u` filtra por usuario (escribir `student`, Enter), `c` muestra la línea de comando completa, `k` pide un PID y luego la señal (Enter = 15), `r` renice, `h` ayuda, `q` sale. Qué decir: Ctrl+C también sale de `top`; **no** mata nada, solo cierra `top`. La cabecera es `uptime` + `free` + resumen de estados, todo en una pantalla.

6. `top` en modo batch, útil para pegar en un ticket o en un script:

```bash
top -b -n 1 | head -12
```
Qué observar: `-b` sin interactividad, `-n 1` una sola muestra. Se puede redirigir a un archivo.

7. Mirar dentro de `/proc` de la propia shell:

```bash
head -8 /proc/$$/status
tr '\0' ' ' < /proc/$$/cmdline; echo
ls -l /proc/$$/exe /proc/$$/cwd
ls -l /proc/$$/fd
```
Salida esperada:
```text
Name:	bash
Umask:	0002
State:	S (sleeping)
Tgid:	2351
Ngid:	0
Pid:	2351
PPid:	2350
TracerPid:	0
-bash
lrwxrwxrwx. 1 student student 0 Sep  4 09:41 /proc/2351/cwd -> /home/student
lrwxrwxrwx. 1 student student 0 Sep  4 09:41 /proc/2351/exe -> /usr/bin/bash
lrwx------. 1 student student 64 Sep  4 09:41 0 -> /dev/pts/0
lrwx------. 1 student student 64 Sep  4 09:41 1 -> /dev/pts/0
lrwx------. 1 student student 64 Sep  4 09:41 2 -> /dev/pts/0
lrwx------. 1 student student 64 Sep  4 09:41 255 -> /dev/pts/0
```
Qué observar: `exe` apunta al binario **real** aunque el proceso se llame de otra forma; `fd` 0, 1 y 2 son stdin/stdout/stderr, todos conectados a la terminal `pts/0`. `sudo ls -l /proc/1/exe` mostraría `/usr/lib/systemd/systemd`. Un proceso del kernel (`sudo ls -l /proc/2/exe`) da "No such file or directory": no tiene binario.

- **Checkpoint:** pegar en el chat la salida de:

```bash
ps -eo pid,ppid,user,stat,cmd --sort=-%cpu | head -4; nproc; uptime
```

### Lab 1.2 — Proceso runaway, señales, jobs, prioridades y carga (20 min)

- **Objetivo:** provocar un proceso que consume CPU, localizarlo y terminarlo con `top` y con `pkill`; practicar jobs, `nohup`, `nice`/`renice` y leer carga y memoria.

1. Lanzar dos procesos que consumen CPU y ver subir la carga:

```bash
yes > /dev/null &
yes > /dev/null &
jobs
sleep 20; uptime
```
Salida esperada:
```text
[1] 3001
[2] 3002
[1]-  Running                 yes > /dev/null &
[2]+  Running                 yes > /dev/null &
 09:45:33 up 43 min,  1 user,  load average: 1.42, 0.45, 0.17
```
Qué observar: `yes` imprime "y" sin parar; enviado a `/dev/null` solo quema CPU. El load average a 1 min sube hacia 2.00 (dos procesos en `R` sobre 2 núcleos = 100 %). Dejarlos correr.

2. Encontrarlos con `top`: ejecutar `top`, pulsar `1` (las dos CPU al ~100 %), `P` (ya está por CPU: los dos `yes` arriba con ~99 %), y matar **uno** desde `top`: pulsar `k`, escribir el PID del primer `yes`, Enter, y en "Send pid 3001 signal [15/sigterm]" pulsar Enter. Ver que desaparece. Salir con `q`.

```bash
top
```
Qué observar: en `top` se ven `%Cpu0` y `%Cpu1` al 100 % `us` y los `yes` con estado `R`. Tras matarlo, la carga empieza a bajar lentamente (es un promedio).

3. Matar el segundo por nombre, mirando antes:

```bash
pgrep -a yes
pkill yes
sleep 1; pgrep -a yes || echo "sin procesos yes"
jobs
```
Salida esperada:
```text
3002 yes
[1]-  Terminated              yes > /dev/null
[2]+  Terminated              yes > /dev/null
sin procesos yes
```
Qué observar: `pkill` envía `TERM` (15) por defecto. La shell avisa "Terminated" de sus jobs. Si un proceso ignora `TERM`, entonces `pkill -9 yes` o `kill -9 PID`, nunca como primera opción.

4. Señales de pausa y continuación, y `kill -l`:

```bash
kill -l | head -3
sleep 600 &
kill -STOP %1; ps -o pid,stat,cmd -p $!
kill -CONT %1; ps -o pid,stat,cmd -p $!
kill %1
```
Salida esperada:
```text
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
Qué observar: `$!` es el PID del último proceso en segundo plano; `%1` es el job 1. `STOP` deja el estado en `T`; `CONT` lo devuelve a `S`. Sirve para "congelar" un respaldo pesado en horario laboral y reanudarlo después.

5. Control de trabajos con Ctrl+Z, `bg` y `fg`:

```bash
sleep 300
```
Pulsar **Ctrl+Z**. Luego:

```bash
jobs
bg %1
jobs
fg %1
```
Pulsar **Ctrl+C** para terminarlo. Salida esperada:
```text
^Z
[1]+  Stopped                 sleep 300
[1]+  Stopped                 sleep 300
[1]+ sleep 300 &
[1]+  Running                 sleep 300 &
sleep 300
^C
```
Qué observar: Ctrl+Z **no** mata, suspende (estado `T`); `bg` lo reanuda atrás; `fg` lo trae al frente, donde Ctrl+C sí lo termina. Error típico: creer que Ctrl+Z cerró el programa y dejar diez `vim` suspendidos; `jobs` lo delata.

6. Sobrevivir al cierre de la sesión con `nohup`:

```bash
nohup sleep 900 > /dev/null 2>&1 &
echo "PID a buscar después de volver a entrar: $!"
exit
```
Volver a entrar con `ssh -p 2222 student@localhost` y comprobar:

```bash
pgrep -a sleep
ps -o pid,ppid,cmd -p $(pgrep -o sleep)
pkill sleep
```
Salida esperada:
```text
3080 sleep 900
    PID    PPID CMD
   3080       1 sleep 900
```
Qué observar: sobrevivió al `exit` y ahora su padre es PID 1 (o el `systemd --user` de student, que también sirve). Sin `nohup`, la shell le habría enviado `HUP` al cerrar. Qué decir: en RHEL 9 los procesos en segundo plano suelen sobrevivir al logout incluso sin `nohup` (`KillUserProcesses=no` en `logind.conf`), pero no hay que confiar en ello; lo correcto para algo permanente es un servicio.

7. Prioridades: dos procesos compitiendo por **un mismo** núcleo, uno con `nice`:

```bash
taskset -c 0 yes > /dev/null &
taskset -c 0 nice -n 10 yes > /dev/null &
sleep 5; ps -o pid,ni,%cpu,cmd -C yes
```
Salida esperada:
```text
[1] 3101
[2] 3102
    PID  NI %CPU CMD
   3101   0 90.5 yes
   3102  10  9.4 yes
```
Qué observar: `taskset -c 0` obliga a ambos a usar la CPU 0, creando competencia; el de `nice 10` recibe ~10 % frente a ~90 %. Sin `taskset`, en una VM de 2 núcleos cada uno tendría su CPU y el `nice` sería invisible. En `top` se ve `PR 30` para `NI 10`.

8. Cambiar la prioridad en caliente con `renice`:

```bash
renice -n 19 -p 3102
renice -n 0 -p 3102
sudo renice -n -5 -p 3102
top -b -n 2 -d 5 | grep -w yes | tail -2
ps -o pid,ni,cmd -C yes
pkill yes
```
Salida esperada:
```text
3102 (process ID) old priority 10, new priority 19
renice: failed to set priority for 3102 (process ID): Permission denied
3102 (process ID) old priority 19, new priority -5
   3101 student   20   0    2320    576    512 R   9.3   0.0   0:38.21 yes
   3102 student   15  -5    2320    576    512 R  90.1   0.0   0:12.44 yes
    PID  NI CMD
   3101   0 yes
   3102  -5 yes
```
Qué observar: un usuario normal puede **empeorar** la prioridad de sus procesos (19) pero no mejorarla, ni siquiera volver a 0; root sí (`-5`), y la proporción de CPU se invierte. Reemplazar `3102` por el PID real de cada uno. Se mide con `top -b` y no con `ps`, porque el `%CPU` de `ps` es el **promedio de toda la vida del proceso** y tarda mucho en reflejar el cambio; el `%CPU` de `top` es el del intervalo (`-d 5`). En la columna `PR` de `top` se ve `15` para `NI -5` (`PR = 20 + NI`).

9. Memoria y actividad del sistema:

```bash
free -m
vmstat 1 5
```
Salida esperada:
```text
               total        used        free      shared  buff/cache   available
Mem:            3713         352        3010           8         350        3129
Swap:           4027           0        4027
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 3082468   2104 356420    0    0    12     3   80  150  0  0 99  0  0
 0  0      0 3082468   2104 356420    0    0     0     0   62  110  0  0 100 0  0
...
```
Qué observar: `available` (3129) es lo que realmente puede usar una aplicación nueva; `free` (3010) es menor porque el kernel usa el resto como caché. Swap en 0 usados: bien. En `vmstat`, la primera línea es el promedio desde el arranque; las siguientes son muestras reales. `si`/`so` distintos de 0 de forma sostenida = falta memoria; `wa` alto = el disco es el cuello de botella; `r` mayor que `nproc` de forma sostenida = falta CPU.

10. Si sobra tiempo: crear un zombie a propósito y verlo, y revisar el perfil de `tuned`:

```bash
bash -c 'sleep 2 & exec sleep 120' &
sleep 3; ps -eo pid,ppid,stat,cmd | grep -w Z
kill %1; sleep 1; ps -eo pid,ppid,stat,cmd | grep -w Z || echo "zombie liberado"
tuned-adm active; tuned-adm recommend
```
Salida esperada:
```text
   3210    3209 Z    [sleep] <defunct>
zombie liberado
Current active profile: virtual-guest
virtual-guest
```
Qué decir: el zombie no se mata (ya está muerto); se corrige o reinicia al **padre** (PPID). `tuned` aplica perfiles de rendimiento del kernel; en una VM el recomendado es `virtual-guest`; `tuned-adm list` muestra los demás (`throughput-performance`, `balanced`) y `sudo tuned-adm profile throughput-performance` lo cambia. Es objetivo del RHCSA; con esto basta. Si sale `tuned-adm: command not found` o "Cannot talk to TuneD daemon": `sudo dnf install -y tuned && sudo systemctl enable --now tuned`.

- **Checkpoint:** pegar en el chat la salida de:

```bash
jobs; pgrep -a yes || echo "sin procesos yes"; uptime; free -m | head -2
```

Debe mostrar sin jobs, "sin procesos yes" y carga bajando.

## Bloque 2 — Servicios con systemd

### Conceptos (15 min)

**Qué es systemd.** El proceso 1 y el gestor de todo lo que arranca después: servicios, montajes, temporizadores, sockets. Reemplazó a los scripts de `/etc/init.d` de SysV (RHEL 6). Habla en términos de **unidades** (units): archivos de texto con extensión que indica el tipo.

| Tipo | Ejemplo | Para qué |
|---|---|---|
| `.service` | `sshd.service`, `httpd.service` | Un demonio o un comando |
| `.target` | `multi-user.target`, `graphical.target` | Un grupo de unidades; el equivalente a los antiguos runlevels |
| `.timer` | `logrotate.timer`, `dnf-makecache.timer` | Ejecutar un `.service` según calendario (alternativa a cron, Día 07) |
| `.socket` | `cockpit.socket`, `systemd-journald.socket` | Escuchar en un puerto y arrancar el servicio solo cuando llega una conexión |
| `.mount` | `boot.mount`, `-.mount` (la raíz) | Un punto de montaje (generados desde `/etc/fstab`, Día 06) |
| `.path`, `.device`, `.slice`, `.scope` | | Vigilar rutas, hardware, grupos de recursos, sesiones |

**Dónde viven.** `/usr/lib/systemd/system/` (las que instalan los paquetes; no editar), `/etc/systemd/system/` (las del administrador; **ganan** sobre las anteriores con el mismo nombre), `/run/systemd/system/` (generadas en caliente). `systemctl cat unidad` muestra el archivo real y sus drop-ins; `systemctl show unidad` muestra las 200 propiedades efectivas.

**Dos preguntas independientes.** Este es el concepto del día:
- ¿Está corriendo **ahora**? → `start`/`stop`/`restart`/`reload`, se consulta con `is-active` o en la línea `Active:` de `status`.
- ¿Arrancará **con el sistema**? → `enable`/`disable`, se consulta con `is-enabled` o en la línea `Loaded:`.

Un servicio puede estar activo y deshabilitado (funciona hasta el próximo reinicio: el clásico "ayer funcionaba y hoy tras el reinicio no"), o habilitado e inactivo (alguien lo detuvo a mano). `enable --now` hace ambas cosas. Técnicamente `enable` solo crea un enlace simbólico en `/etc/systemd/system/multi-user.target.wants/` apuntando al unit file; `disable` lo borra. Eso es todo.

**`reload` vs `restart`.** `reload` pide al servicio releer su configuración sin cortar conexiones (envía `ExecReload`, normalmente `SIGHUP`); no todos lo soportan. `restart` lo detiene y lo arranca: cambia el `MainPID` y corta clientes. `sshd` soporta `reload` y además `restart` no cierra las sesiones ya abiertas (`KillMode=process`), pero por costumbre: **nunca `stop sshd` por SSH**; si ocurre, se entra por la consola del hipervisor.

**`mask`.** Va más allá de `disable`: crea un enlace `/etc/systemd/system/nombre.service → /dev/null`, de modo que nadie puede arrancar la unidad, ni a mano ni como dependencia de otra. Se usa para servicios que deben quedar prohibidos (por ejemplo, un `firewalld` reemplazado por otra solución, o un servicio conflictivo). `unmask` deshace. Detalle que sorprende: no se puede enmascarar una unidad cuyo archivo real está en `/etc/systemd/system/` (mask quiere poner ahí su enlace y el archivo ya existe).

**`status` se lee completo:**
```text
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Thu 2026-09-04 10:12:01 EST; 5s ago
   Main PID: 4321 (httpd)
      Tasks: 177 (limit: 23000)
     Memory: 24.1M
     CGroup: /system.slice/httpd.service
             ├─4321 /usr/sbin/httpd -DFOREGROUND
   ... últimas 10 líneas del journal de la unidad ...
```
`Loaded:` dice de qué archivo viene, si está habilitado y qué haría el *preset* de RHEL por defecto. `Active:` puede ser `active (running)`, `active (exited)` (un `oneshot` que terminó bien), `inactive (dead)`, `failed`, `activating`. `CGroup:` lista todos los procesos que systemd considera parte del servicio (por eso `stop` los mata a todos, aunque hayan hecho fork). Abajo, el journal: la mitad de los diagnósticos terminan ahí.

**Targets.** `multi-user.target` es el "runlevel 3": red, servicios, consola de texto. `graphical.target` lo incluye y agrega el escritorio. `systemctl get-default`/`set-default` fijan cuál arranca; `systemctl isolate target` cambia en caliente (no ejecutar por SSH salvo hacia `multi-user`). `rescue.target` (root sin red, un solo usuario) y `emergency.target` (ni siquiera monta `/` en escritura) son las herramientas del Día 10 para arrancar un servidor roto.

**Anatomía de un unit file de servicio:**
```ini
[Unit]            # qué es y con qué se relaciona
Description=...
After=network.target        # orden: arrancar después de (no lo exige)
Requires= / Wants=          # dependencia dura / blanda

[Service]         # cómo se ejecuta
Type=simple       # el proceso de ExecStart ES el servicio (default). Otros: forking, oneshot, notify, exec
ExecStart=/ruta/al/programa   # ruta absoluta obligatoria
ExecReload=...
Restart=on-failure           # reiniciar si termina mal (también: always, on-abnormal, no)
RestartSec=5
User=                        # con qué cuenta (default root)

[Install]         # qué hace enable
WantedBy=multi-user.target   # enable crea el enlace en multi-user.target.wants/
```
Después de crear o editar un unit file: `sudo systemctl daemon-reload`, siempre. systemd avisa si se olvida.

**Hora del sistema.** Los logs se ordenan y correlacionan por hora; con la hora mal, no se puede investigar nada (ni comparar con el firewall, ni con el directorio). `chronyd` sincroniza con NTP (`chronyc sources`, `chronyc tracking`); `timedatectl` muestra zona horaria, si hay sincronización y permite fijarlas. En Panamá: `America/Panama` (UTC-5, sin horario de verano).

### Lab 2.1 — Controlar servicios: sshd, httpd, targets, timers y sockets (20 min)

- **Objetivo:** leer el estado de los servicios, instalar `httpd` y administrarlo de principio a fin (start, enable, reload, mask), inspeccionar dependencias y targets.

1. Panorama: qué corre, qué está habilitado, qué falló:

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled | head -15
systemctl --failed
```
Salida esperada (resumida):
```text
  UNIT                     LOAD   ACTIVE SUB     DESCRIPTION
  auditd.service           loaded active running Security Auditing Service
  chronyd.service          loaded active running NTP client/server
  crond.service            loaded active running Command Scheduler
  NetworkManager.service   loaded active running Network Manager
  rsyslog.service          loaded active running System Logging Service
  sshd.service             loaded active running OpenSSH server daemon
  systemd-journald.service loaded active running Journal Service
  ...
UNIT FILE                  STATE   PRESET
auditd.service             enabled enabled
chronyd.service            enabled enabled
crond.service              enabled enabled
...
  UNIT LOAD ACTIVE SUB DESCRIPTION
0 loaded units listed.
```
Qué observar: `list-units` habla del **ahora** (`ACTIVE`); `list-unit-files` habla del **arranque** (`STATE`: enabled, disabled, static = sin `[Install]`, no se habilita porque otra la arranca, masked). `--failed` vacío es lo deseado; si hay algo, `systemctl status esa.unidad` y `journalctl -u`.

2. Radiografía de `sshd`:

```bash
systemctl status sshd
systemctl cat sshd
systemctl show sshd -p MainPID -p ActiveState -p SubState -p UnitFileState -p FragmentPath
systemctl is-active sshd; systemctl is-enabled sshd; systemctl is-failed sshd
```
Salida esperada (extractos):
```text
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-09-04 09:02:40 EST; 1h 5min ago
   Main PID: 890 (sshd)
...
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
MainPID=890
ActiveState=active
SubState=running
UnitFileState=enabled
FragmentPath=/usr/lib/systemd/system/sshd.service
active
enabled
active
```
Qué observar: `ExecReload` es un `kill -HUP` al PID principal: eso es "reload". `KillMode=process` explica por qué un `restart sshd` no cierra nuestra sesión. `Restart=on-failure` con 42 s. `is-failed` responde `active` cuando no ha fallado (y `failed` si falló); los tres `is-*` devuelven además un código de salida útil en scripts. Diferencia RHEL 8 → 9 que conviene mencionar: en RHEL 8 esta unidad traía `EnvironmentFile=-/etc/crypto-policies/back-ends/opensshserver.config` y `$CRYPTO_POLICY` en el `ExecStart`; en RHEL 9 la política criptográfica se aplica desde la propia configuración, con `Include /etc/crypto-policies/back-ends/opensshserver.config` dentro de `/etc/ssh/sshd_config.d/50-redhat.conf` (Día 05). ⚠️ Verificar en la VM antes de la clase: la salida exacta de `systemctl cat sshd` en la 9.x instalada.

3. Instalar Apache y observar su estado inicial (el paquete no lo arranca ni lo habilita: política de RHEL):

```bash
sudo dnf install -y httpd
systemctl status httpd
```
Salida esperada:
```text
...
Installed:
  apr-1.7.0-12.el9.x86_64  apr-util-1.6.1-23.el9.x86_64  httpd-2.4.57-11.el9_4.x86_64  httpd-core-... httpd-filesystem-... httpd-tools-... mod_http2-... mod_lua-... redhat-logos-httpd-...
Complete!
○ httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; preset: disabled)
     Active: inactive (dead)
       Docs: man:httpd.service(8)
```
Qué observar: `disabled` e `inactive`: instalado no significa nada más. Las versiones exactas dependen de la 9.x instalada.

4. Arrancar, comprobar, y ver que **no** quedó habilitado:

```bash
sudo systemctl start httpd
systemctl is-active httpd; systemctl is-enabled httpd
echo "<h1>rhel01 - Dia 04</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```
Salida esperada:
```text
active
disabled
<h1>rhel01 - Dia 04</h1>
<h1>rhel01 - Dia 04</h1>
```
Qué observar: funciona, pero un reinicio lo dejaría apagado. Desde el navegador del equipo propio (`http://localhost:8080`) **no** carga todavía: el firewall de la VM solo permite SSH y cockpit; se abre el Día 08. Pregunta para la clase: ¿qué comando falta?

5. Habilitar, y ver qué hizo `enable` en realidad:

```bash
sudo systemctl enable httpd
ls -l /etc/systemd/system/multi-user.target.wants/httpd.service
systemctl is-enabled httpd
```
Salida esperada:
```text
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
lrwxrwxrwx. 1 root root 37 Sep  4 10:14 /etc/systemd/system/multi-user.target.wants/httpd.service -> /usr/lib/systemd/system/httpd.service
enabled
```
Qué observar: un enlace simbólico, nada más. `disable` lo borra. Cuando `multi-user.target` arranca, arranca todo lo que hay en su carpeta `.wants/`.

6. `reload` mantiene el PID, `restart` lo cambia:

```bash
systemctl show -p MainPID --value httpd
sudo systemctl reload httpd; systemctl show -p MainPID --value httpd
sudo systemctl restart httpd; systemctl show -p MainPID --value httpd
```
Salida esperada:
```text
4321
4321
4410
```
Qué observar: tras cambiar la configuración de Apache en producción se usa `reload` (graceful): los clientes no se enteran. `restart` corta conexiones. Si un servicio no tiene `ExecReload`, `systemctl reload` dice "Job type reload is not applicable".

7. Enmascarar y desenmascarar:

```bash
sudo systemctl stop httpd
sudo systemctl mask httpd
sudo systemctl start httpd
ls -l /etc/systemd/system/httpd.service
systemctl is-enabled httpd
sudo systemctl unmask httpd
sudo systemctl enable --now httpd
systemctl is-active httpd; systemctl is-enabled httpd
```
Salida esperada:
```text
Created symlink /etc/systemd/system/httpd.service → /dev/null.
Failed to start httpd.service: Unit httpd.service is masked.
lrwxrwxrwx. 1 root root 9 Sep  4 10:18 /etc/systemd/system/httpd.service -> /dev/null
masked
Removed "/etc/systemd/system/httpd.service".
active
enabled
```
Qué observar: el mask es el enlace a `/dev/null` en `/etc/systemd/system/`, que tiene prioridad sobre `/usr/lib/systemd/system/`. `mask` no detiene un servicio en ejecución (por eso el `stop` previo, o `mask --now`). Este es exactamente el ticket 2 del reto.

8. Dependencias: quién necesita a quién:

```bash
systemctl list-dependencies httpd | head -12
systemctl list-dependencies --reverse httpd
```
Salida esperada:
```text
httpd.service
● ├─httpd-init.service
● ├─system.slice
● ├─basic.target
● │ ├─...
● └─sysinit.target
...
httpd.service
● └─multi-user.target
●   └─graphical.target
```
Qué observar: la lista exacta varía según la 9.x y los paquetes instalados; lo importante es la forma. `httpd-init.service` (genera el certificado autofirmado la primera vez) solo aparece si está instalado `mod_ssl`, que no entra con `dnf install httpd`: si no sale, no es un error. Al revés, `multi-user.target` quiere a `httpd` porque está habilitado. `list-dependencies multi-user.target` muestra todo lo que arranca en un servidor.

9. Targets:

```bash
systemctl get-default
systemctl list-units --type=target | head -12
sudo systemctl set-default graphical.target
sudo systemctl set-default multi-user.target
ls -l /etc/systemd/system/default.target
```
Salida esperada:
```text
multi-user.target
  UNIT                   LOAD   ACTIVE SUB    DESCRIPTION
  basic.target           loaded active active Basic System
  cryptsetup.target      loaded active active Local Encrypted Volumes
  getty.target           loaded active active Login Prompts
  multi-user.target      loaded active active Multi-User System
  network-online.target  loaded active active Network is Online
  network.target         loaded active active Network
  ...
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/graphical.target.
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/multi-user.target.
lrwxrwxrwx. 1 root root 41 Sep  4 10:20 /etc/systemd/system/default.target -> /usr/lib/systemd/system/multi-user.target
```
Qué observar: `default.target` es otro enlace simbólico. En nuestra VM sin escritorio, `graphical.target` arrancaría igual en texto (no hay display manager), pero en el examen RHCSA piden "que arranque en multi-user/graphical por defecto" y esto es la respuesta. `systemctl isolate rescue.target` se verá el Día 10 desde la consola.

10. Otros tipos de unidad en acción: timers, sockets y mounts:

```bash
systemctl list-timers
systemctl list-units --type=socket | head -8
systemctl list-units --type=mount
```
Salida esperada (extractos):
```text
NEXT                        LEFT       LAST                        PASSED  UNIT                         ACTIVATES
Thu 2026-09-04 11:00:00 EST 38min left Thu 2026-09-04 10:00:00 EST 21min ago dnf-makecache.timer          dnf-makecache.service
Fri 2026-09-05 00:00:00 EST 13h left   Thu 2026-09-04 09:02:41 EST 1h ago    logrotate.timer              logrotate.service
Fri 2026-09-05 09:17:12 EST 22h left   Thu 2026-09-04 09:17:12 EST 1h ago    systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Mon 2026-09-07 00:00:00 EST 2 days left ...                                   fstrim.timer                 fstrim.service
...
  UNIT                     LOAD   ACTIVE SUB       DESCRIPTION
  cockpit.socket           loaded active listening Cockpit Web Service Socket
  dbus.socket              loaded active running   D-Bus System Message Bus Socket
  systemd-journald.socket  loaded active running   Journal Socket
...
  UNIT              LOAD   ACTIVE SUB     DESCRIPTION
  -.mount           loaded active mounted Root Mount
  boot.mount        loaded active mounted /boot
  ...
```
Qué observar: `logrotate` y `dnf makecache` en RHEL 9 corren por **timer**, no por cron (`systemctl cat logrotate.timer` muestra `OnCalendar=daily`). `cockpit.socket` (si aparece: el instalador de RHEL 9 lo habilita en la instalación "Server") escucha en el 9090 y solo arranca `cockpit.service` cuando alguien se conecta: activación por socket. Los `.mount` se generan desde `/etc/fstab` (Día 06).

- **Checkpoint:** pegar en el chat la salida de:

```bash
systemctl is-active httpd; systemctl is-enabled httpd; systemctl get-default; curl -s http://localhost
```

Debe mostrar `active`, `enabled`, `multi-user.target` y `<h1>rhel01 - Dia 04</h1>`.

### Lab 2.2 — Unidad propia `monitor.service` y hora del sistema (15 min)

- **Objetivo:** escribir un servicio desde cero que registra la carga cada 30 s, habilitarlo, verlo reiniciarse solo tras un fallo, y comprobar la sincronización de hora.

1. El script que ejecutará el servicio. Escribe dos veces: a su salida estándar (systemd la captura al journal) y a syslog con `logger` (facilidad `local0`, que usaremos en el Bloque 3):

```bash
sudo tee /usr/local/bin/monitor.sh > /dev/null <<'EOF'
#!/bin/bash
# monitor.sh - registra carga y memoria cada 30 s (curso Dia 04)
INTERVALO=${INTERVALO:-30}
while true; do
    CARGA=$(cut -d' ' -f1-3 /proc/loadavg)
    MEM=$(free -m | awk '/^Mem:/ {print $3"/"$2" MB"}')
    echo "carga=${CARGA} mem_usada=${MEM}"
    logger -t monitor -p local0.info "carga=${CARGA} mem_usada=${MEM}"
    sleep "$INTERVALO"
done
EOF
sudo chmod 755 /usr/local/bin/monitor.sh
ls -lZ /usr/local/bin/monitor.sh
INTERVALO=1 timeout 3 /usr/local/bin/monitor.sh
```
Salida esperada:
```text
-rwxr-xr-x. 1 root root unconfined_u:object_r:bin_t:s0 340 Sep  4 10:25 /usr/local/bin/monitor.sh
carga=0.12 0.20 0.15 mem_usada=390/3713 MB
carga=0.12 0.20 0.15 mem_usada=390/3713 MB
carga=0.12 0.20 0.15 mem_usada=390/3713 MB
```
Qué observar: se prueba el script **antes** de convertirlo en servicio (`timeout 3` lo corta a los 3 s). `ls -Z` mostrará el contexto SELinux `bin_t`, correcto porque se creó directamente en `/usr/local/bin`; si se hubiera creado en `/home` y movido con `mv`, systemd no podría ejecutarlo (Día 08).

2. El unit file:

```bash
sudo tee /etc/systemd/system/monitor.service > /dev/null <<'EOF'
[Unit]
Description=Monitor de carga y memoria (curso Dia 04)
Documentation=file:/usr/local/bin/monitor.sh
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemd-analyze verify /etc/systemd/system/monitor.service
sudo systemctl daemon-reload
```
Qué observar: `systemd-analyze verify` no imprime nada si el archivo es correcto; con un error de sintaxis (por ejemplo `ExecStart` con ruta relativa) lo describe. `daemon-reload` es obligatorio para que systemd lea el archivo nuevo.

3. Habilitar, arrancar y verificar:

```bash
sudo systemctl enable --now monitor.service
systemctl status monitor
```
Salida esperada:
```text
Created symlink /etc/systemd/system/multi-user.target.wants/monitor.service → /etc/systemd/system/monitor.service.
● monitor.service - Monitor de carga y memoria (curso Dia 04)
     Loaded: loaded (/etc/systemd/system/monitor.service; enabled; preset: disabled)
     Active: active (running) since Thu 2026-09-04 10:27:02 EST; 8s ago
       Docs: file:/usr/local/bin/monitor.sh
   Main PID: 5120 (monitor.sh)
      Tasks: 2 (limit: 23000)
     Memory: 1.1M
        CPU: 14ms
     CGroup: /system.slice/monitor.service
             ├─5120 /bin/bash /usr/local/bin/monitor.sh
             └─5133 sleep 30

Sep 04 10:27:02 rhel01 systemd[1]: Started Monitor de carga y memoria (curso Dia 04).
Sep 04 10:27:02 rhel01 monitor.sh[5120]: carga=0.10 0.18 0.14 mem_usada=392/3713 MB
```
Qué observar: `preset: disabled` es normal para unidades propias (RHEL solo pre-habilita las suyas). En el `CGroup` se ven el script y su `sleep`: los dos son "el servicio". La línea de `monitor.sh[5120]` es la salida estándar capturada por el journal.

4. Seguir el log en vivo y con filtros (Ctrl+C para salir de `-f`):

```bash
journalctl -u monitor -n 3 --no-pager
journalctl -u monitor -f
```
Salida esperada:
```text
Sep 04 10:27:02 rhel01 systemd[1]: Started Monitor de carga y memoria (curso Dia 04).
Sep 04 10:27:02 rhel01 monitor.sh[5120]: carga=0.10 0.18 0.14 mem_usada=392/3713 MB
Sep 04 10:27:32 rhel01 monitor.sh[5120]: carga=0.08 0.17 0.14 mem_usada=392/3713 MB
```
Qué observar: `-u` filtra por unidad, `-n 3` últimas tres líneas, `--no-pager` para pegar en el chat, `-f` sigue en vivo como `tail -f`.

5. Simular un fallo y ver `Restart=on-failure` en acción:

```bash
sudo kill -9 $(systemctl show -p MainPID --value monitor)
sleep 7; systemctl status monitor --no-pager | grep -E 'Active|Main PID'
journalctl -u monitor -n 6 --no-pager | grep systemd
```
Salida esperada:
```text
     Active: active (running) since Thu 2026-09-04 10:29:10 EST; 2s ago
   Main PID: 5210 (monitor.sh)
Sep 04 10:29:05 rhel01 systemd[1]: monitor.service: Main process exited, code=killed, status=9/KILL
Sep 04 10:29:05 rhel01 systemd[1]: monitor.service: Failed with result 'signal'.
Sep 04 10:29:10 rhel01 systemd[1]: monitor.service: Scheduled restart job, restart counter is at 1.
Sep 04 10:29:10 rhel01 systemd[1]: Stopped Monitor de carga y memoria (curso Dia 04).
Sep 04 10:29:10 rhel01 systemd[1]: Started Monitor de carga y memoria (curso Dia 04).
```
Qué observar: murió con señal 9 (fallo), systemd esperó `RestartSec=5` y lo relanzó con un PID nuevo. Un `systemctl stop` limpio **no** dispara el reinicio: eso distingue `on-failure` de `always`. Esto es lo que un `nohup` jamás dará.

6. Hora del sistema: NTP y zona horaria:

```bash
timedatectl
chronyc sources
chronyc tracking | head -4
```
Salida esperada:
```text
               Local time: Thu 2026-09-04 10:31:12 EST
           Universal time: Thu 2026-09-04 15:31:12 UTC
                 RTC time: Thu 2026-09-04 15:31:12
                Time zone: America/Panama (EST, -0500)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 129.6.15.28                   1   6   377    35  -1201us[-1357us] +/-   27ms
^+ 200.29.147.31                 2   6   377    36   +850us[ +850us] +/-   61ms
^- ntp.example.net               2   6   377    33  +4521us[+4521us] +/-   74ms
Reference ID    : 81060F1C (129.6.15.28)
Stratum         : 2
Ref time (UTC)  : Thu Sep 04 15:30:41 2026
System time     : 0.000312 seconds fast of NTP time
```
Qué observar: `System clock synchronized: yes` y `NTP service: active` es lo que hay que ver en todo servidor. Los nombres/IP de las fuentes **serán distintos en cada VM**: la configuración por defecto de RHEL 9 (`/etc/chrony.conf`) trae una sola línea `pool 2.rhel.pool.ntp.org iburst`, y chrony resuelve varios servidores de ese pool. En `chronyc sources`, `^*` marca la fuente elegida, `^+` las candidatas, `^-` las descartadas, `^?` inalcanzables (típico en la institución si el firewall bloquea UDP 123 hacia afuera: entonces se configura un servidor NTP interno en `/etc/chrony.conf` con `server ntp.pgn.local iburst`). Si la zona horaria no fuera Panamá: `sudo timedatectl set-timezone America/Panama` (`timedatectl list-timezones | grep -i panama` para el nombre exacto); si NTP estuviera apagado: `sudo timedatectl set-ntp true` (activa `chronyd`).

- **Checkpoint:** pegar en el chat la salida de:

```bash
systemctl is-active monitor; systemctl is-enabled monitor; journalctl -u monitor -n 1 --no-pager; timedatectl | grep -E 'Time zone|synchronized'
```

## Bloque 3 — Logs del sistema

### Conceptos (10 min)

**Dos sistemas que conviven.** En RHEL 9 los mensajes fluyen así: los servicios escriben (por `/dev/log`, por su salida estándar, o directamente) → **`systemd-journald`** los recibe todos, les añade metadatos (unidad, PID, UID, binario) y los guarda en formato binario indexado → **`rsyslog`** los lee del journal (`imjournal`) y escribe los archivos de texto clásicos en `/var/log` según reglas. Se consultan con `journalctl` (potente, filtrable, pero por defecto puede no sobrevivir al reinicio) y con `less`/`grep`/`tail` sobre `/var/log` (texto plano, persistente, rotado por `logrotate`). Un administrador usa los dos.

**`/var/log`, los archivos que hay que conocer:**

| Archivo | Contenido | Quién escribe |
|---|---|---|
| `messages` | Todo lo de nivel `info` o superior salvo correo, autenticación y cron. El primero que se abre | rsyslog |
| `secure` | Autenticación y autorización: `sshd`, `sudo`, `su`, `passwd`, PAM | rsyslog (`authpriv.*`) |
| `cron` | Ejecuciones de cron/anacron (Día 07) | rsyslog |
| `boot.log` | Mensajes de arranque de los servicios (las líneas `[  OK  ]`) | systemd/plymouth lo escriben; rsyslog además dirige ahí `local7.*` |
| `maillog`, `spooler` | Correo, uucp/news (casi vacíos) | rsyslog |
| `dnf.log`, `dnf.rpm.log`, `hawkey.log` | Historial de dnf: qué se instaló y cuándo | dnf |
| `audit/audit.log` | Auditoría del kernel: SELinux (`AVC`), llamadas vigiladas, logins. Solo root. Se lee con `ausearch` (Día 08) | auditd |
| `wtmp`, `btmp`, `lastlog` | Binarios: `last`, `lastb`, `lastlog` | login, sshd |
| `httpd/access_log`, `error_log` | Apache (recién instalado) | httpd |
| `sssd/`, `samba/`, `chrony/`, `firewalld` | Un directorio o archivo por servicio | cada servicio |

**Regla de rsyslog: `facilidad.prioridad  destino`.** La facilidad dice **quién** (`auth`, `authpriv`, `cron`, `daemon`, `kern`, `mail`, `syslog`, `user`, `local0`–`local7` para uso propio) y la prioridad **cuán grave**, de menor a mayor: `debug`, `info`, `notice`, `warning`, `err`, `crit`, `alert`, `emerg`. `*.info` significa "info **o más grave**". `mail.none` excluye. Las reglas viven en `/etc/rsyslog.conf` (las de RHEL) y `/etc/rsyslog.d/*.conf` (las nuestras; se incluyen antes de las reglas del archivo principal). Ejemplos:
```text
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                  /var/log/secure
local0.*                                    /var/log/monitor.log
*.emerg                                     :omusrmsg:*        # a la pantalla de todos
```
`logger` inyecta un mensaje con la facilidad y prioridad que uno quiera: es la herramienta para probar reglas y para que los scripts (`backup.sh`, Día 07) escriban en el log del sistema.

**`journalctl`: los filtros que se usan a diario.** `-u unidad`, `-f` (seguir), `-b` (este arranque), `-b -1` (el anterior: para investigar por qué se reinició), `-p err` (prioridad `err` o peor), `--since "1 hour ago"` / `--since today` / `--since "2026-09-04 09:00"` / `--until`, `-t identificador` (por etiqueta, ej. `sudo`), `-k` (solo kernel, como `dmesg`), `-xe` (con explicaciones y al final: el comando que systemd sugiere cuando un servicio falla), `-o verbose` (todos los campos), `_PID=`, `_UID=`, `_COMM=` (campos directos), `-g patrón` (grep), `-n N`, `-r` (inverso), `--no-pager`, `--disk-usage`, `--list-boots`.

**Persistencia del journal.** `Storage=auto` en `/etc/systemd/journald.conf` significa: si existe `/var/log/journal/`, escribe ahí (persistente); si no, en `/run/log/journal/` (RAM, se pierde al reiniciar). La instalación estándar de RHEL 9 puede venir de cualquiera de las dos formas según la versión y el medio: **hay que comprobarlo**. El RHCSA lo pide explícitamente ("preserve system journals").

**`logrotate`.** Los archivos de `/var/log` crecen sin límite si nadie los corta. `logrotate` los rota (renombra con fecha, comprime, borra los viejos) según `/etc/logrotate.conf` (defaults: `weekly`, `rotate 4`, `create`, `dateext`) y un archivo por servicio en `/etc/logrotate.d/`. En RHEL 9 lo dispara `logrotate.timer` a diario; `logrotate -d` simula sin tocar nada.

### Lab 3.1 — rsyslog, logger, accesos fallidos y journalctl (12 min)

- **Objetivo:** leer las reglas de rsyslog, crear una propia para `local0`, generar eventos reales de seguridad y encontrarlos con `journalctl` y en `/var/log`.

1. Las reglas activas de rsyslog y dónde se incluyen las nuestras:

```bash
grep -vE '^\s*(#|$)' /etc/rsyslog.conf
ls /etc/rsyslog.d/
```
Salida esperada (extracto de la sección de reglas):
```text
module(load="imuxsock" SysSock.Use="off")
module(load="imjournal" StateFile="imjournal.state")
...
include(file="/etc/rsyslog.d/*.conf" mode="optional")
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
authpriv.*                                              /var/log/secure
mail.*                                                  -/var/log/maillog
cron.*                                                  /var/log/cron
*.emerg                                                 :omusrmsg:*
uucp,news.crit                                          /var/log/spooler
local7.*                                                /var/log/boot.log
```
Qué observar: `imjournal` = rsyslog lee del journal. El `-` delante de `/var/log/maillog` significa escritura sin `sync` (más rápido). `/etc/rsyslog.d/` está vacío en una instalación limpia.

2. Regla propia: la facilidad `local0` (la que usa `monitor.service`) a su propio archivo, y que no se duplique en `messages`:

```bash
sudo tee /etc/rsyslog.d/monitor.conf > /dev/null <<'EOF'
# Mensajes de la facilidad local0 (monitor.service y scripts propios)
local0.*    /var/log/monitor.log
& stop
EOF
sudo rsyslogd -N1
sudo systemctl restart rsyslog
logger -p local0.notice -t prueba "hola desde logger (local0)"
logger "mensaje sin opciones: facilidad user, prioridad notice"
sudo tail -2 /var/log/monitor.log
sudo tail -1 /var/log/messages
```
Salida esperada:
```text
rsyslogd: version 8.2102.0-117.el9, config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
Sep  4 11:02:15 rhel01 monitor[5210]: carga=0.05 0.09 0.10 mem_usada=395/3713 MB
Sep  4 11:02:31 rhel01 prueba[5480]: hola desde logger (local0)
Sep  4 11:02:31 rhel01 student[5481]: mensaje sin opciones: facilidad user, prioridad notice
```
Qué observar: `rsyslogd -N1` valida la sintaxis antes de reiniciar (equivalente a `visudo -c`). `& stop` significa "lo que coincidió con la regla anterior, no sigas procesándolo": sin esa línea, `local0.info` también cumpliría `*.info` y aparecería en `messages`. El mensaje de `logger` sin opciones va con la etiqueta del usuario a `messages`. `monitor.service` ya está escribiendo en `/var/log/monitor.log` cada 30 s.

3. Generar eventos de seguridad reales: tres intentos de SSH con un usuario inexistente contra la propia VM (escribir cualquier contraseña tres veces; a la pregunta de la huella del host responder `yes`):

```bash
ssh intruso@localhost
```
Salida esperada:
```text
The authenticity of host 'localhost (::1)' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
intruso@localhost's password:
Permission denied, please try again.
intruso@localhost's password:
Permission denied, please try again.
intruso@localhost's password:
intruso@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
```

4. Encontrar el rastro en los tres lugares donde queda:

```bash
sudo grep 'Failed password' /var/log/secure | tail -3
sudo lastb | head -4
sudo journalctl -u sshd --since "5 min ago" -g 'Failed|Invalid' --no-pager
```
Salida esperada:
```text
Sep  4 11:05:02 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
Sep  4 11:05:05 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
Sep  4 11:05:08 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
intruso  ssh:notty    ::1              Thu Sep  4 11:05 - 11:05  (00:00)
intruso  ssh:notty    ::1              Thu Sep  4 11:05 - 11:05  (00:00)
intruso  ssh:notty    ::1              Thu Sep  4 11:05 - 11:05  (00:00)
Sep 04 11:05:00 rhel01 sshd[5510]: Invalid user intruso from ::1 port 47122
Sep 04 11:05:02 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
...
```
Qué observar: la misma información en `secure` (texto, `authpriv`), en `btmp` (`lastb`, binario) y en el journal (filtrado por unidad, tiempo y patrón). En un servidor expuesto, `sudo lastb | wc -l` da miles: es la razón de `fail2ban` o de restringir SSH en el firewall (Día 08). `::1` es localhost en IPv6.

5. Batería de filtros de `journalctl` que hay que dominar:

```bash
sudo journalctl -b -p err --no-pager | tail -5
sudo journalctl -b -1 -n 3 --no-pager
sudo journalctl --since today -t sudo -n 3 --no-pager
sudo journalctl -k -n 3 --no-pager
sudo journalctl _PID=1 -n 2 --no-pager
sudo journalctl -u monitor -o verbose -n 1 --no-pager | grep -E '_SYSTEMD_UNIT|_PID|_COMM|MESSAGE=|PRIORITY'
sudo journalctl -xe --no-pager | tail -5
```
Salida esperada (extractos):
```text
Sep 04 09:02:38 rhel01 kernel: ... (errores del arranque, si los hay; puede estar vacío)
Specifying boot ID or boot offset has no effect, no persistent journal was found.
   (o, si el journal ya es persistente, las últimas 3 líneas del arranque anterior)
Sep 04 10:14:02 rhel01 sudo[4300]:  student : TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/systemctl enable httpd
...
Sep 04 09:02:36 rhel01 kernel: Linux version 5.14.0-427.13.1.el9_4.x86_64 ...
Sep 04 10:29:10 rhel01 systemd[1]: Started Monitor de carga y memoria (curso Dia 04).
    PRIORITY=6
    _PID=5210
    _COMM=monitor.sh
    _SYSTEMD_UNIT=monitor.service
    MESSAGE=carga=0.05 0.09 0.10 mem_usada=395/3713 MB
```
Qué observar: `-p err` en el arranque actual debe estar casi vacío en una VM sana. Si `-b -1` dice "no persistent journal was found", el journal es volátil y **se perdió todo lo anterior al último reinicio**: se corrige en el Lab 3.2. `-o verbose` muestra los campos con los que se puede filtrar directamente (`_SYSTEMD_UNIT=`, `_COMM=`, `_UID=`). `-xe` es lo que systemd sugiere cuando un `start` falla: va al final y añade explicaciones.

- **Checkpoint:** pegar en el chat la salida de:

```bash
sudo tail -1 /var/log/monitor.log; sudo lastb | head -1; sudo journalctl -u sshd --since "10 min ago" -g Failed --no-pager | wc -l
```

### Lab 3.2 — Journal persistente y logrotate (8 min)

- **Objetivo:** dejar el journal en disco para que sobreviva reinicios y configurar la rotación de `/var/log/monitor.log`.

1. Comprobar la situación actual:

```bash
ls -ld /var/log/journal 2>/dev/null || echo "NO existe /var/log/journal: journal volátil en /run/log/journal"
journalctl --disk-usage
journalctl --list-boots
grep -E '^#?Storage' /etc/systemd/journald.conf
```
Salida esperada (caso volátil):
```text
NO existe /var/log/journal: journal volátil en /run/log/journal
Archived and active journals take up 16.0M in the file system.
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
  0 3f2a...                          Thu 2026-09-04 09:02:36 EST Thu 2026-09-04 11:10:02 EST
#Storage=auto
```
Qué observar: un solo arranque en la lista y sin directorio en `/var/log`: volátil. Si el directorio ya existe con un subdirectorio de 32 caracteres (el machine-id) y `system.journal` dentro, ya es persistente: pasar al paso 3 y verificar igual.

2. Hacerlo persistente. Dos formas; se usa la explícita (sobrevive a cualquier duda sobre `auto`):

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/persistente.conf > /dev/null <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=200M
EOF
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
ls -l /var/log/journal/
journalctl --disk-usage
```
Salida esperada:
```text
total 0
drwxr-sr-x+ 2 root systemd-journal 60 Sep  4 11:12 8c1f0e2a5b3d4c6e9f0a1b2c3d4e5f60
Archived and active journals take up 8.0M in the file system.
```
Qué observar: el drop-in en `journald.conf.d/` evita editar el archivo del paquete (misma técnica que `sudoers.d` y `rsyslog.d`). `systemd-tmpfiles --create` pone permisos y ACL correctos (grupo `systemd-journal`, lectura para `adm` y `wheel`: por eso `student` puede leer el journal sin `sudo`). `SystemMaxUse` limita el tamaño (default: 10 % del sistema de archivos, máximo 4 GB). Tras el reinicio de la tarea, `journalctl --list-boots` mostrará `-1` y `0`, y `-b -1` funcionará. Alternativa equivalente: solo `mkdir /var/log/journal` + `restart` con `Storage=auto`.

3. Rotación: leer la configuración global y la de rsyslog, y simular:

```bash
grep -vE '^\s*(#|$)' /etc/logrotate.conf
cat /etc/logrotate.d/rsyslog
sudo logrotate -d /etc/logrotate.conf 2>&1 | grep -A3 'messages'
```
Salida esperada:
```text
weekly
rotate 4
create
dateext
include /etc/logrotate.d
/var/log/cron
/var/log/maillog
/var/log/messages
/var/log/secure
/var/log/spooler
{
    missingok
    sharedscripts
    postrotate
        /usr/bin/systemctl -s HUP kill rsyslog.service >/dev/null 2>&1 || true
    endscript
}
rotating pattern: /var/log/cron /var/log/maillog /var/log/messages ... weekly (4 rotations)
...
considering log /var/log/messages
  log does not need rotating (log has been rotated at 2026-9-1 0:0, that is not week ago yet)
```
Qué observar: semanal, 4 copias, archivo nuevo tras rotar, sufijo con fecha (`messages-20260901`). `postrotate` manda `HUP` a rsyslog para que abra el archivo nuevo: si no, seguiría escribiendo en el viejo renombrado. `-d` es *debug*: simula y explica, no toca nada.

4. Rotación para nuestro `/var/log/monitor.log`, forzada para ver el resultado:

```bash
sudo tee /etc/logrotate.d/monitor > /dev/null <<'EOF'
/var/log/monitor.log {
    daily
    rotate 7
    dateext
    compress
    delaycompress
    missingok
    notifempty
    create 0600 root root
    postrotate
        /usr/bin/systemctl -s HUP kill rsyslog.service >/dev/null 2>&1 || true
    endscript
}
EOF
sudo logrotate -d /etc/logrotate.d/monitor 2>&1 | tail -3
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor.log*
sleep 31; sudo tail -1 /var/log/monitor.log
```
Salida esperada:
```text
considering log /var/log/monitor.log
  Now: 2026-09-04 11:15
  log does not need rotating ...
-rw-------. 1 root root    0 Sep  4 11:15 /var/log/monitor.log
-rw-------. 1 root root 1840 Sep  4 11:15 /var/log/monitor.log-20260904
Sep  4 11:15:40 rhel01 monitor[5210]: carga=0.03 0.06 0.08 mem_usada=396/3713 MB
```
Qué observar: `-f` fuerza la rotación aunque no toque; el archivo viejo lleva la fecha (`dateext` puesto explícitamente porque al ejecutar un archivo de `logrotate.d` suelto no se leen los defaults globales); `delaycompress` deja sin comprimir la copia más reciente (rsyslog podría estar escribiéndola); tras el `HUP`, rsyslog escribe en el archivo nuevo. `/var/lib/logrotate/logrotate.status` guarda cuándo se rotó cada uno.

- **Checkpoint:** pegar en el chat la salida de:

```bash
ls -d /var/log/journal/*/ ; journalctl --disk-usage; ls /var/log/monitor.log*
```

## Bloque 4 — Gestión de software: RPM, DNF y repositorios

### Conceptos (15 min)

**Paquete RPM.** Un archivo `.rpm` es un contenedor firmado con: los archivos a instalar y dónde van, metadatos (versión, licencia, descripción), la lista de **dependencias** (qué otros paquetes o librerías necesita) y *scriptlets* (comandos que corren antes/después de instalar o quitar: crear un usuario de servicio, avisar a systemd). El nombre lo dice todo:

```text
httpd-2.4.57-11.el9_4.x86_64.rpm
 │     │      │   │     └── arquitectura: x86_64, aarch64, noarch (scripts, datos)
 │     │      │   └──────── dist tag: el9 = RHEL 9; el9_4 = actualización publicada para 9.4
 │     │      └──────────── release: iteración del empaquetado de Red Hat (parches, backports)
 │     └─────────────────── version: la del proyecto original (upstream)
 └───────────────────────── name
```
Qué decir: en RHEL la **versión upstream casi no cambia** durante los 10 años de vida de la release (Apache seguirá siendo 2.4.57 en RHEL 9); lo que cambia es el `release`, porque Red Hat aplica los parches de seguridad sin cambiar de versión (*backporting*). Por eso un escáner de vulnerabilidades que solo mira "2.4.57" da falsos positivos: se comprueba con `rpm -q --changelog httpd | grep CVE-...`.

**La base de datos RPM** (`/var/lib/rpm`) registra cada paquete instalado y cada archivo que puso. `rpm -q...` la consulta: `-qa` todos, `-qi` información, `-ql` archivos, `-qc` solo los de configuración, `-qd` documentación, `-qf /ruta` a qué paquete pertenece un archivo, `--changelog`, `--scripts`, `-V` verifica que los archivos no cambiaron (integridad: `S` tamaño, `5` hash, `T` fecha, `c` = archivo de configuración, donde cambiar es normal). Con `-p` las mismas consultas sobre un archivo `.rpm` sin instalar (`-qpi`, `-qpl`).

**`rpm -i` / `rpm -e` existen, pero casi no se usan.** `rpm -ivh paquete.rpm` instala un archivo; `rpm -e nombre` desinstala. El problema: `rpm` **no resuelve dependencias** (falla con "Failed dependencies" y hay que buscarlas a mano) y no queda en el historial de `dnf`. Se ven para entender la capa de abajo y para el caso de un `.rpm` suelto en un servidor sin red. Nunca `--nodeps` ni `--force` en producción.

**DNF** (`yum` en RHEL 9 es un enlace a `dnf`: `ls -l /usr/bin/yum`) es la capa que resuelve dependencias, descarga de repositorios, verifica firmas y registra transacciones. Sus verbos: `search`, `info`, `list`, `provides`, `install`, `remove`, `autoremove`, `update`/`upgrade` (sinónimos), `check-update`, `history` (con `info N`, `undo N`, `redo N`), `group`, `module`, `repolist`, `clean`, `makecache`, `download` (plugin). Configuración en `/etc/dnf/dnf.conf` (`gpgcheck=1`, `installonly_limit=3`: cuántos kernels conserva, `clean_requirements_on_remove=True`). Paquetes protegidos en `/etc/dnf/protected.d/` (nunca deja quitar `dnf`, `systemd`, `sudo`, el kernel en uso).

**Repositorios.** Un repositorio es un directorio (local, HTTP, HTTPS) con paquetes y un subdirectorio `repodata/` con los índices. Se definen en `/etc/yum.repos.d/*.repo`:
```ini
[id-del-repo]
name=Nombre legible
baseurl=https://... | file:///ruta
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-...
```
En RHEL, `/etc/yum.repos.d/redhat.repo` lo genera y regenera `subscription-manager` a partir de la suscripción: **no se edita**; los repos de Red Hat se activan con `subscription-manager repos --enable`. Los importantes:
- **BaseOS**: el sistema operativo (kernel, systemd, bash, coreutils). Soporte completo.
- **AppStream**: aplicaciones y lenguajes (httpd, php, nodejs, podman). Soporte completo. Algunos vienen en varias versiones como **módulos** (`nodejs:18`, `:20`, `:22`): se elige el *stream*.
- **CodeReady Linux Builder (CRB)**: librerías de desarrollo; sin soporte; deshabilitado por defecto. EPEL lo necesita.
- **EPEL** (Extra Packages for Enterprise Linux): repositorio comunitario del proyecto Fedora con miles de paquetes que Red Hat no incluye (`htop`, `fail2ban`, `nginx` extras...). Riesgos: no lo soporta Red Hat, puede quedar sin mantenedor, y una actualización desde EPEL puede reemplazar un paquete de RHEL si hay conflictos de nombre. Regla institucional razonable: instalado pero **deshabilitado** (`enabled=0`) y activado por comando (`--enablerepo=epel`) para paquetes concretos.

**Firmas GPG: por qué importan.** Cada RPM de Red Hat va firmado con la llave privada de Red Hat; la pública está en `/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release` e importada en la base RPM (`rpm -q gpg-pubkey`). Con `gpgcheck=1`, `dnf` rechaza cualquier paquete cuya firma no verifique: protege contra un espejo comprometido o un ataque en la red, incluso sin HTTPS. Al usar un repositorio nuevo (EPEL, DVD), `dnf` pide importar su llave la primera vez: hay que leer la huella (`Fingerprint`) y compararla con la publicada antes de aceptar. `gpgcheck=0` solo se justifica para paquetes propios sin firmar, y con conciencia del riesgo.

**Servidor sin internet (el caso de la institución).** Opciones, de simple a completa: (1) montar la ISO DVD y usarla como repositorio local (`baseurl=file:///mnt/dvd/BaseOS`): hoy; (2) un servidor interno que sincroniza los repos de Red Hat (`reposync`) y los publica por HTTP (`httpd` + `createrepo_c`): todos los servidores apuntan a él; (3) Red Hat Satellite: la versión soportada de (2) con gestión de parches y suscripciones. Para un `.rpm` propio o de terceros: `dnf install ./archivo.rpm` (resuelve dependencias desde los repos disponibles) y, si son muchos, un directorio con `createrepo_c`.

### Lab 4.1 — Consultar, instalar, quitar y deshacer (20 min)

- **Objetivo:** dominar las consultas `rpm -q*`, el ciclo `dnf search/info/install/remove`, instalar un `.rpm` con `rpm -ivh` entendiendo sus límites, y deshacer una transacción con `dnf history undo`.

1. `yum` es `dnf`; estado de los repositorios y de la base RPM:

```bash
ls -l /usr/bin/yum /usr/bin/dnf
sudo dnf repolist
rpm -qa | wc -l
rpm -qa --last | head -3
```
Salida esperada:
```text
lrwxrwxrwx. 1 root root 5 ... /usr/bin/dnf -> dnf-3
lrwxrwxrwx. 1 root root 5 ... /usr/bin/yum -> dnf-3
Updating Subscription Management repositories.
repo id                                   repo name
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
612
httpd-2.4.57-11.el9_4.x86_64                  Thu 04 Sep 2026 10:12:40 AM EST
httpd-core-2.4.57-11.el9_4.x86_64             Thu 04 Sep 2026 10:12:40 AM EST
...
```
Qué observar: si en vez de la tabla aparece `This system is not registered with an entitlement server` o `There are no enabled repositories`, la VM no está registrada: `sudo subscription-manager register` (Día 01) y repetir. En UTM los ids llevan `aarch64`. `--last` ordena por fecha de instalación: lo último fue `httpd`.

2. Consultas sobre paquetes instalados:

```bash
rpm -qi bash | head -14
rpm -ql tree
rpm -qc openssh-server
rpm -qd tree
rpm -qf /usr/bin/ls /etc/ssh/sshd_config /usr/local/bin/monitor.sh
rpm -q --changelog bash | head -4
rpm -q --scripts openssh-server | head -8
rpm -V openssh-server; echo "rpm -V terminó con código $?"
```
Salida esperada (extractos):
```text
Name        : bash
Version     : 5.1.8
Release     : 9.el9
Architecture: x86_64
Install Date: Mon 01 Sep 2026 10:02:11 AM EST
Group       : Unspecified
Size        : 7738298
License     : GPLv3+
Signature   : RSA/SHA256, Tue 13 Feb 2024 ..., Key ID 199e2f91fd431d51
Source RPM  : bash-5.1.8-9.el9.src.rpm
...
/usr/bin/tree
/usr/lib/.build-id
/usr/lib/.build-id/...
/usr/share/doc/tree
/usr/share/doc/tree/LICENSE
/usr/share/doc/tree/README
/usr/share/man/man1/tree.1.gz
/etc/pam.d/sshd
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/50-redhat.conf
/etc/sysconfig/sshd
/usr/share/doc/tree/LICENSE
/usr/share/doc/tree/README
/usr/share/man/man1/tree.1.gz
coreutils-8.32-35.el9.x86_64
openssh-server-8.7p1-38.el9.x86_64
file /usr/local/bin/monitor.sh is not owned by any package
* Tue Feb 13 2024 Siteshwar Vashisht <svashisht@redhat.com> - 5.1.8-9
- Fix CVE-2022-3715
...
preinstall scriptlet (using /bin/sh):
getent group sshd >/dev/null || groupadd -g 74 -r sshd || :
getent passwd sshd >/dev/null || useradd -c "Privilege-separated SSH" -u 74 -g sshd -s /sbin/nologin -r -d /usr/share/empty.sshd sshd 2> /dev/null || :
postinstall scriptlet (using /bin/sh):
if [ $1 -eq 1 ] && [ -x "/usr/lib/systemd/systemd-update-helper" ]; then
    /usr/lib/systemd/systemd-update-helper install-system-units sshd.service sshd.socket || :
fi
rpm -V terminó con código 0
```
Qué observar: `Signature ... Key ID 199e2f91fd431d51` es la llave de Red Hat (release key 2). `-qc` responde "¿qué archivos de este paquete puedo editar?": la pregunta que uno se hace al instalar algo nuevo. Nótese `/etc/ssh/sshd_config.d/50-redhat.conf`: en RHEL 9 la configuración de sshd se parte en fragmentos y el Día 05 se toca ese directorio, no el archivo principal. `-qf` es la respuesta a "¿de dónde salió este binario?" (ticket 3 del reto): `monitor.sh` no es de ningún paquete, como cabe esperar. `--changelog` documenta los CVE corregidos por backport. `--scripts` muestra que el paquete crea el usuario `sshd` al instalarse. `-V` sin salida = ningún archivo alterado; el Día 05 se edita `sshd_config` y aparecerá `S.5....T.  c /etc/ssh/sshd_config`.

3. Buscar, informarse e instalar:

```bash
dnf search wget
dnf info wget | head -12
sudo dnf install -y wget
rpm -q wget
```
Salida esperada (extractos):
```text
===================== Name Exactly Matched: wget =====================
wget.x86_64 : A utility for retrieving files using the HTTP or FTP protocols
Available Packages
Name         : wget
Version      : 1.21.1
Release      : 8.el9_4
Architecture : x86_64
Size         : 790 k
Source       : wget-1.21.1-8.el9_4.src.rpm
Repository   : rhel-9-for-x86_64-appstream-rpms
Summary      : A utility for retrieving files using the HTTP or FTP protocols
...
Dependencies resolved.
==============================================================================
 Package   Arch     Version           Repository                          Size
==============================================================================
Installing:
 wget      x86_64   1.21.1-8.el9_4    rhel-9-for-x86_64-appstream-rpms   790 k

Transaction Summary
==============================================================================
Install  1 Package
...
Installed:
  wget-1.21.1-8.el9_4.x86_64
Complete!
wget-1.21.1-8.el9_4.x86_64
```
Qué observar: `search` busca en nombre y resumen; `info` dice de qué repositorio viene y cuánto pesa **antes** de instalar; `install -y` responde sí a todo (en clase; en producción, leer primero el "Transaction Summary", sobre todo la sección "Removing" si aparece).

4. Quitar `tree` con `dnf` y volver a instalarlo desde el archivo `.rpm` con `rpm` (para entender la capa de abajo):

```bash
sudo dnf remove -y tree
cd /tmp && dnf download tree
ls -l tree-*.rpm
rpm -qpi tree-*.rpm | grep -E '^(Name|Version|Release|Signature)'
rpm -qpl tree-*.rpm | head -3
rpm -K tree-*.rpm
sudo rpm -ivh tree-*.rpm
rpm -q tree
cd ~
```
Salida esperada:
```text
...
Removed:
  tree-1.8.0-10.el9.x86_64
Complete!
tree-1.8.0-10.el9.x86_64.rpm                    1.3 MB/s |  56 kB     00:00
-rw-r--r--. 1 student student 57344 Sep  4 11:32 tree-1.8.0-10.el9.x86_64.rpm
Name        : tree
Version     : 1.8.0
Release     : 10.el9
Signature   : RSA/SHA256, ..., Key ID 199e2f91fd431d51
/usr/bin/tree
/usr/lib/.build-id
/usr/lib/.build-id/...
tree-1.8.0-10.el9.x86_64.rpm: digests signatures OK
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:tree-1.8.0-10.el9                ################################# [100%]
tree-1.8.0-10.el9.x86_64
```
Qué observar: `-qp` consulta el archivo sin instalarlo; `rpm -K` verifica la firma contra las llaves importadas (`digests signatures OK`: intacto y firmado por Red Hat). `-ivh` = install, verbose, hash (barra de progreso). Funcionó porque `tree` no tiene dependencias que falten; con `httpd` habría fallado con "Failed dependencies: apr...". Qué decir: la forma correcta de instalar un `.rpm` suelto es `sudo dnf install ./tree-1.8.0-10.el9.x86_64.rpm`: resuelve dependencias y queda en el historial. En UTM el archivo se llama `tree-1.8.0-10.el9.aarch64.rpm`; por eso el comodín.

5. El historial de transacciones y deshacer una:

```bash
dnf history | head -8
dnf history info 1 | head -12
dnf history list wget
```
Localizar en esa última salida el **ID** de la transacción cuya "Command line" es `install -y wget` (en el ejemplo, `13`) y deshacerla escribiendo ese número a mano:
```bash
sudo dnf history undo -y 13
rpm -q wget
dnf history | head -4
```
Salida esperada (los ID varían):
```text
ID     | Command line                    | Date and time    | Action(s)      | Altered
-----------------------------------------------------------------------------------
    14 | remove -y tree                  | 2026-09-04 11:31 | Removed        |    1
    13 | install -y wget                 | 2026-09-04 11:28 | Install        |    1
    12 | install -y httpd                | 2026-09-04 10:12 | Install        |   10
    11 | install -y tree vim-enhanced... | 2026-09-02 09:40 | Install        |    6
    ...
     1 |                                 | 2026-09-01 09:55 | Install        |  520 E
Transaction ID : 1
Begin time     : Mon 01 Sep 2026 09:55:02 AM EST
...
User           : System <unset>
Return-Code    : Success
Releasever     : 9
Command Line   :
Comment        :
Packages Altered:
    Install NetworkManager-1:1.46.0-4.el9.x86_64 @anaconda
...
Removing:
 wget      x86_64   1.21.1-8.el9_4   @rhel-9-for-x86_64-appstream-rpms   3.1 M
...
Removed:
  wget-1.21.1-8.el9_4.x86_64
Complete!
package wget is not installed
ID     | Command line                    | Date and time    | Action(s)      | Altered
-----------------------------------------------------------------------------------
    15 | history undo -y 13              | 2026-09-04 11:34 | Removed        |    1
    14 | remove -y tree                  | 2026-09-04 11:31 | Removed        |    1
```
Qué observar: la transacción 1 es la instalación del sistema (`@anaconda`, ~520 paquetes); la letra al final de la columna "Altered" es una marca de dnf (`E` = la transacción terminó con advertencias o errores no fatales en la salida; `>` = la base RPM cambió por fuera de dnf después; `*` = abortada). **Los ID son distintos en cada VM: hay que leerlos, no copiar el 13 del ejemplo.** Cada `dnf history info N` dice quién, cuándo, qué comando y qué paquetes con `@repositorio`: es la respuesta al ticket 3 del reto. `undo` genera una transacción nueva (no borra la anterior): el historial es auditable. Nótese que el `rpm -ivh` de `tree` **no** aparece: rpm no pasa por dnf. `dnf history list wget` acota a las transacciones que tocaron ese paquete.

6. ¿Qué paquete me da tal comando o archivo? `provides`, con y sin éxito:

```bash
dnf provides /usr/bin/ss
dnf provides '*/htop'
dnf provides '*/nc'
```
Salida esperada:
```text
iproute-6.2.0-5.el9.x86_64 : Advanced IP routing and network device configuration tools
Repo        : rhel-9-for-x86_64-baseos-rpms
Matched from:
Filename    : /usr/bin/ss
Error: No Matches found
nmap-ncat-3:7.92-1.el9.x86_64 : Nmap's Netcat replacement
Repo        : rhel-9-for-x86_64-appstream-rpms
Matched from:
Filename    : /usr/bin/nc
```
Qué observar: `provides` responde "quiero el comando `ss`, ¿qué instalo?" (respuesta: `iproute`). `htop` no existe en BaseOS ni AppStream: lo resolvemos con EPEL en el Lab 4.2. El `3:` delante de `7.92` es el *epoch*, un número que fuerza el orden de versiones.

7. Actualizaciones, grupos y módulos (lectura; instalar un módulo queda opcional):

```bash
dnf check-update > /tmp/updates.txt; echo "código de salida: $?"
head -6 /tmp/updates.txt
dnf group list | head -12
dnf module list nodejs
```
Salida esperada (extractos):
```text
código de salida: 100
Updating Subscription Management repositories.
Last metadata expiration check: 0:05:12 ago on Thu 04 Sep 2026 11:30:02 AM EST.

kernel.x86_64                     5.14.0-503.11.1.el9_5      rhel-9-for-x86_64-baseos-rpms
openssl.x86_64                    1:3.2.2-6.el9_5            rhel-9-for-x86_64-baseos-rpms
python3-libs.x86_64               3.9.21-1.el9_5             rhel-9-for-x86_64-appstream-rpms
Available Environment Groups:
   Server with GUI
   Minimal Install
   Workstation
   ...
Installed Environment Groups:
   Server
Installed Groups:
   Headless Management
Available Groups:
   Container Management
   ...
Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
Name     Stream   Profiles                                 Summary
nodejs   18       common [d], development, minimal, s2i    Javascript runtime
nodejs   20       common [d], development, minimal, s2i    Javascript runtime
nodejs   22       common [d], development, minimal, s2i    Javascript runtime

Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled
```
Qué observar: `check-update` devuelve `100` si hay actualizaciones y `0` si no (útil en scripts); en una VM instalada hace días **siempre habrá** actualizaciones, así que lo normal es ver `100` y una lista larga (por eso se guarda en un archivo: si se hiciera `dnf check-update | head`, el `$?` sería el de `head`, no el de `dnf`). **No ejecutar `dnf update` en clase**: se llevaría 15–20 min y puede traer un kernel nuevo; `sudo dnf update -y` las aplicaría (`upgrade` es lo mismo) y queda como tarea opcional del participante. Los grupos empaquetan conjuntos (`dnf group info "System Tools"` los detalla; `sudo dnf group install "Headless Management"` instalaría Cockpit si no estuviera). Los módulos ofrecen varias versiones del mismo software en el mismo repo: `sudo dnf module install nodejs:20` instala Node 20 (`module reset nodejs` vuelve al estado inicial; `dnf install nodejs` sin módulo instala la versión no modular por defecto). Streams disponibles según la 9.x.

8. Caché de metadatos:

```bash
sudo dnf clean all
sudo dnf makecache
sudo du -sh /var/cache/dnf
```
Salida esperada:
```text
Updating Subscription Management repositories.
38 files removed
Updating Subscription Management repositories.
Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)   35 MB/s |  45 MB     00:01
Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)      30 MB/s |  20 MB     00:00
Metadata cache created.
70M	/var/cache/dnf
```
Qué observar: `clean all` borra metadatos y paquetes descargados; `makecache` los vuelve a bajar. Es el primer remedio cuando `dnf` se queja de metadatos corruptos o de un paquete que "existe pero no se encuentra".

- **Checkpoint:** pegar en el chat la salida de:

```bash
rpm -q tree wget httpd; dnf history | head -3; rpm -qf /usr/bin/ss
```

### Lab 4.2 — Repositorios Red Hat, CodeReady Builder y EPEL (10 min)

- **Objetivo:** ver cómo se definen los repositorios, habilitar CRB con `subscription-manager`, instalar EPEL verificando su llave, instalar `htop` y aprender a dejar EPEL bajo control.

1. Los repositorios definidos y su origen:

```bash
ls /etc/yum.repos.d/
head -12 /etc/yum.repos.d/redhat.repo
sudo subscription-manager repos --list-enabled
sudo dnf repolist --all | grep -iE 'codeready|baseos|appstream|supplementary' | head -8
```
Salida esperada (extractos):
```text
redhat.repo
#
# Certificate-Based Repositories
# Managed by (rhsm) subscription-manager
#
# *** This file is auto-generated.  Changes made here will be over-written. ***
# *** Use "subscription-manager repo-override --help" if you wish to make changes. ***
#

[rhel-9-for-x86_64-baseos-rpms]
name = Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
baseurl = https://cdn.redhat.com/content/dist/rhel9/$releasever/x86_64/baseos/os
+----------------------------------------------------------+
    Available Repositories in /etc/yum.repos.d/redhat.repo
+----------------------------------------------------------+
Repo ID:   rhel-9-for-x86_64-baseos-rpms
Repo Name: Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
Repo URL:  https://cdn.redhat.com/content/dist/rhel9/$releasever/x86_64/baseos/os
Enabled:   1

Repo ID:   rhel-9-for-x86_64-appstream-rpms
...
codeready-builder-for-rhel-9-x86_64-rpms        Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)   disabled
rhel-9-for-x86_64-appstream-rpms                Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)   enabled
rhel-9-for-x86_64-baseos-rpms                   Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)      enabled
rhel-9-for-x86_64-supplementary-rpms            Red Hat Enterprise Linux 9 for x86_64 - Supplementary (RPMs) disabled
```
Qué observar: el archivo lo genera `subscription-manager` (no se edita); la suscripción Developer da acceso a decenas de repos, casi todos deshabilitados. `$releasever` se expande a `9` (o a `9.4` si se fija con `subscription-manager release --set=9.4` para congelar una versión menor, práctica común en instituciones).

2. Habilitar CodeReady Builder con `subscription-manager` (la forma correcta para repos de Red Hat):

```bash
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
sudo dnf repolist
```
Salida esperada:
```text
Repository 'codeready-builder-for-rhel-9-x86_64-rpms' is enabled for this system.
Updating Subscription Management repositories.
repo id                                   repo name
codeready-builder-for-rhel-9-x86_64-rpms  Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
```
Qué observar: `$(arch)` escribe `x86_64` o `aarch64` según la VM: el mismo comando sirve en VirtualBox y en UTM. EPEL depende de librerías que están en CRB; sin él, algunos paquetes de EPEL fallan por dependencias.

3. Instalar EPEL desde su URL oficial y revisar qué trajo:

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
ls /etc/yum.repos.d/
grep -E '^\[|^enabled|^gpgkey' /etc/yum.repos.d/epel.repo | head -6
rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}  %{SUMMARY}\n'
```
Salida esperada:
```text
...
Installing:
 epel-release      noarch   9-7.el9      @commandline    19 k
...
Complete!
epel.repo  epel-testing.repo  redhat.repo
[epel]
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-9
[epel-debuginfo]
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-9
gpg-pubkey-fd431d51-4ae0493b  gpg(Red Hat, Inc. (release key 2) <security@redhat.com>)
gpg-pubkey-5a6340b3-6229229e  gpg(Red Hat, Inc. (auxiliary key 3) <security@redhat.com>)
```
Qué observar: `@commandline` = se instaló desde un archivo/URL, no desde un repo. `epel-release` solo trae los `.repo` y la llave pública en `/etc/pki/rpm-gpg/`, pero la llave **todavía no está importada** en la base RPM (solo aparecen las dos de Red Hat). Puede aparecer también `epel-cisco-openh264.repo`: es normal.

4. Instalar `htop`: la primera instalación desde EPEL pide importar la llave:

```bash
sudo dnf install -y htop
rpm -q gpg-pubkey --qf '%{NAME}  %{SUMMARY}\n' | grep -i epel
dnf info htop | grep -E '^(Name|Version|Repository|From repo)'
```
Salida esperada:
```text
Extra Packages for Enterprise Linux 9 - x86_64              2.1 MB/s |  24 MB     00:11
Dependencies resolved.
==============================================================================
 Package    Arch      Version          Repository     Size
==============================================================================
Installing:
 htop       x86_64    3.3.0-1.el9      epel          185 k
...
Extra Packages for Enterprise Linux 9 - x86_64              1.6 MB/s | 1.6 kB     00:00
Importing GPG key 0x3228467C:
 Userid     : "Fedora (epel9) <epel@fedoraproject.org>"
 Fingerprint: FF8A D134 4597 106E CE81 3B91 8A38 72BF 3228 467C
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-9
Key imported successfully
...
Installed:
  htop-3.3.0-1.el9.x86_64
Complete!
gpg-pubkey-3228467c-613798eb  gpg(Fedora (epel9) <epel@fedoraproject.org>)
Name         : htop
Version      : 3.3.0
Repository   : @System
From repo    : epel
```
Qué observar: con `-y` la llave se importa sin preguntar; sin `-y`, `dnf` muestra la huella y espera `y`: ese es el momento de compararla con la publicada en `https://fedoraproject.org/security/` (o en la documentación de EPEL, `https://docs.fedoraproject.org/en-US/epel/`). `From repo : epel` es la respuesta al ticket 3. Probar `htop` medio minuto: `F6` ordenar, `F5` árbol, `F9` matar, `F3` buscar, `q` salir.

5. Dejar EPEL bajo control: deshabilitado por defecto, activado por comando:

```bash
sudo dnf config-manager --set-disabled epel
dnf repolist --all | grep -E '^epel '
dnf info fail2ban 2>&1 | tail -1
dnf --enablerepo=epel info fail2ban | grep -E '^(Name|Repository)'
sudo dnf config-manager --set-enabled epel
dnf repolist | grep -c epel
```
Salida esperada:
```text
epel                        Extra Packages for Enterprise Linux 9 - x86_64   disabled
Error: No matching Packages to list
Name         : fail2ban
Repository   : epel
1
```
Qué observar: `config-manager` (paquete `dnf-plugins-core`) edita `enabled=` en el `.repo`. Con EPEL deshabilitado, `fail2ban` "no existe"; con `--enablerepo=epel` solo para ese comando, sí. Lo dejamos habilitado en la VM del curso; en un servidor institucional la recomendación es dejarlo en `0`. Para repos de Red Hat se usa `subscription-manager repos --enable/--disable`; `config-manager` es para los de terceros.

- **Checkpoint:** pegar en el chat la salida de:

```bash
dnf repolist | awk 'NR>1 {print $1}'; rpm -q htop epel-release; dnf info htop | grep 'From repo'
```

### Lab 4.3 — Repositorio local desde la ISO DVD (10 min)

- **Objetivo:** reproducir el caso de un servidor sin acceso a internet: usar la ISO de instalación como repositorio local, con verificación de firmas, e instalar desde ella.

1. Conectar la ISO a la VM **sin apagarla**:
   - VirtualBox: menú de la ventana de la VM → *Devices* → *Optical Drives* → *Choose a disk file...* → seleccionar la ISO de RHEL 9 (la misma del Día 01).
   - UTM: en la barra de la ventana de la VM, icono de unidades (*Drive image options*) → la unidad *CD/DVD* → *Change* → seleccionar la ISO.

   Verificar que el kernel la vio:

```bash
lsblk
sudo dmesg | tail -2
sudo blkid /dev/sr0
```
Salida esperada:
```text
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk
├─sda1          8:1    0    1G  0 part /boot
└─sda2          8:2    0   19G  0 part
  ├─rhel-root 253:0    0   15G  0 lvm  /
  └─rhel-swap 253:1    0    4G  0 lvm  [SWAP]
sr0            11:0    1 10.6G  0 rom
[ 8123.401] sr 2:0:0:0: [sr0] scsi3-mmc drive: 32x/32x cd/rw xa/form2 tray
[ 8123.402] cdrom: Uniform CD-ROM driver Revision: 3.20
/dev/sr0: BLOCK_SIZE="2048" UUID="2024-04-04-12-15-33-00" LABEL="RHEL-9-4-0-BaseOS-x86_64" TYPE="iso9660"
```
Qué observar: `sr0` con `RM 1` (removible) y tipo `rom`; `TYPE="iso9660"` es el sistema de archivos de los CD/DVD. `blkid` lleva `sudo` porque `/dev/sr0` es `root:cdrom` con permisos `0660` y `student` no está en el grupo `cdrom`. En UTM el disco principal es `vda`, hay además una partición `/boot/efi` (arranque UEFI) y el lector normalmente también es `sr0`; si no aparece, `lsblk -f | grep iso9660` lo localiza. Si `lsblk` no muestra nada, el hipervisor no conectó la ISO: repetir el paso. ⚠️ Verificar en la VM antes de la clase: el nombre real del lector en UTM.

2. Montar la ISO (solo lectura) y reconocer su estructura:

```bash
sudo mkdir -p /mnt/dvd
sudo mount -o ro /dev/sr0 /mnt/dvd
ls /mnt/dvd
ls /mnt/dvd/BaseOS /mnt/dvd/AppStream
ls /mnt/dvd/BaseOS/Packages | wc -l; ls /mnt/dvd/AppStream/Packages | wc -l
cat /mnt/dvd/media.repo
```
Salida esperada:
```text
mount: /mnt/dvd: WARNING: source write-protected, mounted read-only.
AppStream  BaseOS  EFI  EULA  GPL  RPM-GPG-KEY-redhat-beta  RPM-GPG-KEY-redhat-release  extra_files.json  images  isolinux  media.repo
/mnt/dvd/AppStream:
Packages  repodata
/mnt/dvd/BaseOS:
Packages  repodata
1100
5800
[InstallMedia]
name=Red Hat Enterprise Linux 9.4
mediaid=None
metadata_expire=-1
gpgcheck=0
cost=500
```
Qué observar: dos repositorios completos (`Packages/` + `repodata/`) y la llave pública de Red Hat en la raíz, idéntica a `/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release` (`diff` lo confirma). `media.repo` es la plantilla que sugiere Red Hat; la escribimos completa nosotros. En aarch64 no hay `isolinux/` y los conteos difieren.

3. Definir el repositorio local con verificación de firmas:

```bash
sudo tee /etc/yum.repos.d/rhel9-dvd.repo > /dev/null <<'EOF'
[dvd-baseos]
name=RHEL 9 DVD - BaseOS
baseurl=file:///mnt/dvd/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[dvd-appstream]
name=RHEL 9 DVD - AppStream
baseurl=file:///mnt/dvd/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
sudo dnf repolist
```
Salida esperada:
```text
Updating Subscription Management repositories.
repo id                                   repo name
codeready-builder-for-rhel-9-x86_64-rpms  Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)
dvd-appstream                             RHEL 9 DVD - AppStream
dvd-baseos                                RHEL 9 DVD - BaseOS
epel                                      Extra Packages for Enterprise Linux 9 - x86_64
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
```
Qué observar: `file:///` con tres barras (protocolo `file://` + ruta absoluta). `gpgcheck=1` con la llave de Red Hat: aunque el medio sea local, se verifica que cada paquete esté firmado. En un servidor sin internet, `redhat.repo` no existiría o estaría vacío y este archivo sería el único.

4. Instalar **solo** desde el DVD, como si no hubiera internet, dos paquetes útiles (uno de cada repo):

```bash
sudo dnf --disablerepo="*" --enablerepo="dvd-baseos,dvd-appstream" install -y tmux sysstat
dnf info tmux sysstat | grep -E '^(Name|From repo)'
```
Salida esperada:
```text
Updating Subscription Management repositories.
RHEL 9 DVD - BaseOS                                     45 MB/s | 2.3 MB     00:00
RHEL 9 DVD - AppStream                                  60 MB/s | 8.5 MB     00:00
Dependencies resolved.
==============================================================================
 Package           Arch     Version          Repository       Size
==============================================================================
Installing:
 sysstat           x86_64   12.5.4-7.el9     dvd-appstream   478 k
 tmux              x86_64   3.2a-4.el9       dvd-baseos      480 k
Installing dependencies:
 lm_sensors-libs   x86_64   3.6.0-10.el9     dvd-baseos       42 k
...
Complete!
Name         : sysstat
From repo    : dvd-appstream
Name         : tmux
From repo    : dvd-baseos
```
Qué observar: `--disablerepo="*" --enablerepo="dvd-baseos,dvd-appstream"` fuerza el origen para esta transacción (equivale a `--enablerepo="dvd-*"`); las comillas evitan que la shell expanda el `*`. Las dependencias también salieron del DVD. `tmux` (sesiones de terminal persistentes) y `sysstat` (`iostat`, `mpstat`, `sar`: complemento de `vmstat`) quedan instalados para el resto del curso. ⚠️ Verificar en la VM antes de la clase: cuál de los dos repos del DVD sirve cada paquete (`dnf --disablerepo='*' --enablerepo='dvd-*' info tmux sysstat`); el reparto BaseOS/AppStream cambia entre versiones menores y da igual para el objetivo del lab, pero conviene no leer en clase un `From repo` que no coincide.

5. Dejar el repositorio listo pero inactivo (la ISO no estará montada tras reiniciar) y explicar cómo sería en producción:

```bash
sudo dnf config-manager --set-disabled dvd-baseos dvd-appstream
grep enabled /etc/yum.repos.d/rhel9-dvd.repo
sudo umount /mnt/dvd
```
Salida esperada:
```text
enabled=0
enabled=0
```
Qué decir: en un servidor institucional sin internet el contenido de la ISO se copia al disco (`sudo cp -a /mnt/dvd /srv/rhel9-dvd`) o el archivo `.iso` se deja en el disco y se monta por `/etc/fstab` (`/srv/rhel9.iso /mnt/dvd iso9660 loop,ro,nofail 0 0`, Día 06), el repo queda con `enabled=1` y `redhat.repo` se ignora. **No hacer la copia en la VM del curso:** son unos 10 GB y el disco es de 20 GB; es solo la explicación de cómo se hace en producción. Si el DVD queda habilitado y desmontado, **todo** `dnf` falla con "Failed to download metadata for repo 'dvd-baseos'": por eso se deshabilita ahora. La ISO puede quedar conectada en el hipervisor o expulsarse.

- **Checkpoint:** pegar en el chat la salida de:

```bash
rpm -q tmux sysstat; grep -c 'enabled=0' /etc/yum.repos.d/rhel9-dvd.repo; dnf repolist | wc -l
```

## Reto individual (25 min)

Tres tickets llegan a la mesa de ayuda sobre `rhel01`. El instructor prepara la VM de cada participante con el script de rotura (sección siguiente). El participante resuelve sin pistas y entrega, por ticket: qué estaba mal, cómo lo encontró, cómo lo corrigió y cómo lo verificó. Los tickets 1 y 2 son obligatorios; el 3, para quien termine.

**Ticket #2026-0401 — "El servidor está lento"**
> Usuarios reportan lentitud desde hace unos minutos. Monitoreo muestra la CPU al 50 % sostenido. Identificar el proceso responsable: PID, usuario que lo ejecuta y **binario real** que está corriendo (el nombre que muestra puede engañar). Terminarlo de forma controlada y eliminar el archivo que lo originó. Explicar cómo se distingue de un proceso legítimo del kernel.

**Ticket #2026-0402 — "El monitor dejó de registrar"**
> `/var/log/monitor.log` no recibe entradas desde hace un rato. Dejar `monitor.service` **activo y habilitado** para que arranque con el sistema, y explicar en el reporte qué estado tenía la unidad y qué significa.

**Ticket #2026-0403 — "Software de origen no autorizado"**
> Auditoría detectó el binario `/usr/bin/htop`, que no está en la lista de software aprobado. Indicar: (a) qué paquete lo instaló, (b) desde qué repositorio, (c) en qué transacción de `dnf` y con qué comando. Luego **deshacer exactamente esa transacción** (no simplemente desinstalar) y comprobar que el binario ya no existe.

**Verificación que debe entregar el participante** (pegar en el chat):

```bash
pgrep -a kworkerd || echo "kworkerd: sin procesos"; ls /tmp/kworkerd 2>&1
systemctl is-active monitor; systemctl is-enabled monitor; sudo tail -1 /var/log/monitor.log
rpm -q htop; dnf history | head -3
```

Resultado esperado: "kworkerd: sin procesos" y "No such file or directory"; `active`, `enabled` y una línea de `monitor` con hora reciente; `package htop is not installed` y una transacción `history undo` como la más reciente.

### Script `romper-dia04.sh` (para el instructor)

Los participantes deben ejecutarlo **sin leerlo**. Recomendación: el instructor lo codifica en base64 (`base64 -w0 romper-dia04.sh`) y pega en el chat una sola línea `echo '<base64>' | base64 -d | sudo bash`. Requiere el Lab 2.2 (monitor.service) y el Lab 4.2 (EPEL) completados; si no, crea lo que falte.

```bash
#!/bin/bash
# romper-dia04.sh - prepara los tres tickets del reto del Dia 04. Ejecutar con sudo.
set -u

# Ticket 1: proceso disfrazado que consume CPU, corriendo como pedro
id pedro &>/dev/null || useradd -m -c "Pedro Gomez - Soporte" pedro
cp /usr/bin/yes /tmp/kworkerd
chmod 755 /tmp/kworkerd
sudo -u pedro setsid -f /tmp/kworkerd >/dev/null 2>&1 </dev/null

# Ticket 2: monitor.service "empaquetado" en /usr/lib (como haria un rpm), deshabilitado y enmascarado.
# Se mueve porque systemd no permite enmascarar una unidad cuyo archivo real esta en /etc/systemd/system.
systemctl disable --now monitor.service 2>/dev/null || true
if [ -f /etc/systemd/system/monitor.service ]; then
    mv /etc/systemd/system/monitor.service /usr/lib/systemd/system/monitor.service
fi
systemctl daemon-reload
systemctl mask monitor.service

# Ticket 3: htop instalado desde EPEL (si el Lab 4.2 no se completo, lo instala ahora)
rpm -q htop &>/dev/null || dnf -y -q --enablerepo=epel install htop

echo "Tickets listos: $(date)"
```

Qué deja: un proceso `kworkerd` de `pedro` al 100 % de un núcleo; `monitor.service` con `Loaded: masked` y `Active: inactive`, con el unit file real en `/usr/lib/systemd/system/`; `htop` instalado con su transacción en `dnf history`.

### Solución (para el instructor)

**Ticket 1.** Diagnóstico y corrección:

```bash
top                                   # P: kworkerd ~99 %CPU, USER pedro, estado R; q para salir
ps -eo pid,ppid,user,%cpu,stat,cmd --sort=-%cpu | head -3
PID=$(pgrep -o kworkerd)
sudo ls -l /proc/$PID/exe             # -> /tmp/kworkerd  (un kworker real del kernel no tiene exe)
ps -o pid,ppid,user,cmd -p $PID       # PPID 1 y sin corchetes: no es hilo del kernel (esos tienen PPID 2 y [nombre])
sha256sum /tmp/kworkerd /usr/bin/yes  # mismo hash: es una copia de yes
rpm -qf /tmp/kworkerd                 # not owned by any package
sudo kill $PID; sleep 2; pgrep -a kworkerd || echo eliminado   # TERM basta; -9 solo si no muere
sudo rm -f /tmp/kworkerd
uptime                                # la carga empieza a bajar
```
Detalle que hay que anticipar: `ls -l /proc/PID/exe` **sin `sudo` da "Permission denied"** cuando el proceso es de otro usuario (aquí `pedro`); el enlace `exe` solo lo lee el dueño o root. Si el participante se atasca ahí, la pista es "el proceso no es tuyo".

Puntos a valorar: que use `TERM` antes que `KILL`; que identifique el usuario (`pedro`, no root); que distinga por `/proc/PID/exe` y por el PPID/corchetes; que mencione que un binario en `/tmp` ejecutado por un usuario es un indicador de compromiso y que en un caso real se preservaría el archivo (copiarlo, `stat`, `sudo grep pedro /var/log/secure`) antes de borrarlo.

**Ticket 2.** Diagnóstico y corrección:

```bash
systemctl status monitor              # Loaded: masked (Reason: Unit monitor.service is masked.)  Active: inactive (dead)
ls -l /etc/systemd/system/monitor.service    # -> /dev/null : eso es mask
sudo systemctl unmask monitor         # Removed "/etc/systemd/system/monitor.service".
systemctl cat monitor | head -1       # # /usr/lib/systemd/system/monitor.service (alguien lo "empaqueto")
sudo systemctl enable --now monitor   # Created symlink .../multi-user.target.wants/monitor.service -> /usr/lib/systemd/system/monitor.service
systemctl is-active monitor; systemctl is-enabled monitor
journalctl -u monitor -n 3 --no-pager; sudo tail -1 /var/log/monitor.log
```
Explicación esperada: *mask* = enlace a `/dev/null` en `/etc/systemd/system/` que impide arrancar la unidad por cualquier vía; `disable` solo quita el arranque automático. Error típico: intentar `start` sin `unmask` ("Unit monitor.service is masked") o hacer solo `start` y dejarla `disabled`. Si el participante pregunta por qué el archivo está ahora en `/usr/lib`, la respuesta es la del script: así se ve un servicio instalado por un paquete; funciona igual.

**Ticket 3.** Diagnóstico y corrección:

```bash
rpm -qf /usr/bin/htop                 # htop-3.3.0-1.el9.x86_64
dnf info htop | grep 'From repo'      # From repo : epel
dnf history list htop                 # ID de la transaccion, columna Command line: install -y htop
sudo dnf history info <ID>            # User, Command Line, "Install htop-3.3.0-1.el9.x86_64 @epel"
sudo dnf history undo -y <ID>
rpm -q htop                           # package htop is not installed
ls /usr/bin/htop                      # No such file or directory
dnf history | head -3                 # ultima: history undo <ID>  Removed
```
Alternativas válidas: `dnf history | grep htop`, `rpm -qi htop` para el paquete (no dice el repo: `dnf info` o `history info` sí). No es válido `dnf remove htop` a secas: el ticket pide deshacer la transacción (si la transacción hubiera traído dependencias, `undo` también las quita).

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu \| head` | Los procesos que más CPU consumen (o `--sort=-%mem`) |
| `ps aux`, `ps -ef`, `pstree -p`, `pstree -ps PID` | Todos los procesos (BSD / Unix), árbol, ancestros de un PID |
| `top` (`P` `M` `1` `u` `c` `k` `r` `q`), `top -b -n 1` | Monitor interactivo; modo batch para tickets |
| `pgrep -a patrón`, `pkill patrón`, `pkill -u usuario`, `killall nombre` | Buscar / matar por nombre (mirar con `pgrep` antes) |
| `kill PID`, `kill -9 PID`, `kill -HUP PID`, `kill -STOP/-CONT PID`, `kill -l` | Señales: TERM (default), KILL, recargar, pausar/reanudar, lista |
| `cmd &`, Ctrl+Z, `jobs`, `bg %1`, `fg %1`, `kill %1`, `nohup cmd &` | Control de trabajos de la shell; sobrevivir al logout |
| `nice -n 10 cmd`, `renice -n 5 -p PID`, `taskset -c 0 cmd` | Prioridad al lanzar / en caliente (-20..19); fijar CPU |
| `uptime`, `nproc`, `free -m`, `vmstat 1 5`, `tuned-adm active` | Carga vs núcleos, memoria (`available`), actividad, perfil de rendimiento |
| `ls -l /proc/PID/exe`, `cat /proc/PID/status`, `tr '\0' ' ' < /proc/PID/cmdline` | Binario real, estado, línea de comando de un proceso |
| `systemctl status\|start\|stop\|restart\|reload unidad` | Estado y control del servicio **ahora** |
| `systemctl enable\|disable [--now] unidad`, `is-active`, `is-enabled`, `is-failed` | Arranque automático; consultas rápidas (con código de salida) |
| `systemctl mask\|unmask unidad` | Prohibir / permitir que la unidad arranque (enlace a `/dev/null`) |
| `systemctl list-units --type=service --state=running`, `--failed`, `list-unit-files`, `list-timers` | Qué corre, qué falló, qué está habilitado, temporizadores |
| `systemctl cat unidad`, `systemctl show unidad -p MainPID`, `list-dependencies [--reverse]` | Unit file real, propiedades, dependencias |
| `systemctl get-default`, `set-default multi-user.target`, `isolate` | Target de arranque; cambiar en caliente |
| `sudo systemctl daemon-reload`, `systemd-analyze verify archivo.service` | Tras crear/editar unit files; validar sintaxis |
| `/etc/systemd/system/x.service`: `[Unit] After=` `[Service] Type= ExecStart= Restart=on-failure` `[Install] WantedBy=multi-user.target` | Esqueleto de una unidad propia |
| `timedatectl`, `timedatectl set-timezone America/Panama`, `chronyc sources`, `chronyc tracking` | Hora, zona, sincronización NTP |
| `journalctl -u unidad -f`, `-b`, `-b -1`, `-p err`, `--since "1 hour ago"`, `-t tag`, `-k`, `-xe`, `-o verbose`, `_PID=N`, `-g patrón` | Filtros del journal |
| `journalctl --disk-usage`, `--list-boots`; `Storage=persistent` en `journald.conf.d/` + `mkdir /var/log/journal` | Tamaño y arranques del journal; hacerlo persistente |
| `logger -p local0.notice -t etiqueta "mensaje"` | Escribir en el log del sistema (pruebas, scripts) |
| `/etc/rsyslog.d/x.conf`: `local0.*  /var/log/x.log` + `& stop`; `rsyslogd -N1` | Regla facilidad.prioridad → archivo; validar |
| `/var/log/messages`, `secure`, `cron`, `dnf.log`, `audit/audit.log`; `sudo lastb` | Logs de texto clásicos; intentos fallidos |
| `logrotate -d /etc/logrotate.conf`, `logrotate -f /etc/logrotate.d/x` | Simular / forzar rotación |
| `rpm -qa \| grep x`, `-qi`, `-ql`, `-qc`, `-qd`, `-qf /ruta`, `-q --changelog`, `-q --scripts`, `-V` | Consultar la base RPM |
| `rpm -qpi archivo.rpm`, `rpm -K archivo.rpm`, `sudo rpm -ivh archivo.rpm`, `rpm -e nombre` | Consultar/verificar un `.rpm`; instalar/quitar sin resolver dependencias |
| `dnf search`, `dnf info`, `dnf provides '*/cmd'`, `dnf list installed` | Buscar; qué paquete trae un archivo |
| `sudo dnf install\|remove\|autoremove\|update -y`, `dnf check-update` (código 100 = hay) | Ciclo de vida de paquetes |
| `dnf history`, `history info N`, `history list pkg`, `sudo dnf history undo N` | Transacciones y deshacer |
| `dnf group list\|info\|install "Nombre"`, `dnf module list\|install nodejs:20\|reset nodejs` | Grupos y módulos (AppStream) |
| `dnf repolist [--all]`, `sudo subscription-manager repos --list-enabled\|--enable ID` | Repositorios; los de Red Hat se activan con subscription-manager |
| `sudo dnf config-manager --set-enabled\|--set-disabled ID`, `dnf --enablerepo=epel install x` | Activar/desactivar repos de terceros; por comando |
| `sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm` | EPEL (tras habilitar CRB con `$(arch)`) |
| `sudo mount -o ro /dev/sr0 /mnt/dvd`; `.repo` con `baseurl=file:///mnt/dvd/BaseOS` `gpgcheck=1` `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release` | Repositorio local desde la ISO |
| `sudo dnf clean all`, `sudo dnf makecache`, `rpm -q gpg-pubkey` | Caché de metadatos; llaves GPG importadas |

## Notas para el instructor

- **Preparar antes de la clase:**
  - Restaurar `dia03-fin` en la VM del instructor y ejecutar **todos** los labs en orden, cronometrando. Los que más se desvían: Lab 1.2 (la gente se queda jugando con `top`), Lab 4.1 (ocho pasos en 20 min: los pasos 7 y 8 pasan a demo en cuanto se atrase) y Lab 4.3 (conectar la ISO en el hipervisor).
  - Confirmar en una VM limpia: `rpm -q psmisc dnf-plugins-core tuned tree` (si faltan, avisar al inicio), si `/var/log/journal` existe o no en la 9.x usada (ajustar el discurso del Lab 3.2), y qué streams muestra `dnf module list nodejs`.
  - Contrastar con la VM las salidas que cambian entre versiones menores y anotarlas en el material: `systemctl cat sshd` (en RHEL 9 ya no lleva `$CRYPTO_POLICY`), `systemctl list-dependencies httpd`, `dnf --disablerepo='*' --enablerepo='dvd-*' info tmux sysstat` (de qué repo del DVD sale cada uno) y `chronyc sources` (los servidores del `pool 2.rhel.pool.ntp.org` varían).
  - Confirmar que `htop` **no** está en BaseOS/AppStream de la 9.x instalada (`dnf provides '*/htop'` debe fallar antes de EPEL): si Red Hat lo hubiera añadido, cambiar el ejemplo de EPEL a `fail2ban` o `ncdu`.
  - Descargar por si la red falla: `epel-release-latest-9.noarch.rpm` (con `curl -O`) y un `tree-*.rpm` con `dnf download`, y tenerlos en un lugar para compartir por el chat (`scp -P 2222`).
  - Tener la ISO DVD (no la Boot ISO) localizada en el disco de cada participante: preguntarlo por chat el día anterior. Practicar el menú de conexión de ISO en caliente en VirtualBox y en UTM.
  - Generar el base64 de `romper-dia04.sh` y probar el flujo completo: romper → resolver los tres tickets → verificación.
  - Preparar la respuesta a "¿y cómo instalamos en los servidores de la Procuraduría que no salen a internet?": es el Lab 4.3 más la opción `reposync`/Satellite del bloque de conceptos.
  - Tomar snapshot `dia04-inicio` antes de empezar; `dia04-fin` al terminar.

- **Qué estudiar si es nuevo en RHEL:**
  1. **Unit files y semántica de systemctl:** `man systemd.unit` (secciones `[Unit]`, `[Install]`), `man systemd.service` (`Type=`, `Restart=`, `ExecReload=`), `man systemctl` (`enable`, `mask`, `edit`, `preset`). Practicar en la VM: crear `monitor.service`, matarlo con `-9`, ver el reinicio; intentar `systemctl mask monitor` con el archivo en `/etc/systemd/system` y ver el error "already exists"; moverlo a `/usr/lib` y repetir. Practicar `systemctl edit monitor` (drop-in `override.conf`) y `systemctl revert`.
  2. **journald + rsyslog:** `man journalctl` (sección EXAMPLES), `man journald.conf` (`Storage=`, `SystemMaxUse=`), `man rsyslog.conf` (selectores, `& stop`). Practicar: journal volátil → persistente → reiniciar → `journalctl -b -1`. Verificar que `logger -p local0.info` cae en `/var/log/monitor.log` y **no** en `messages`.
  3. **dnf a fondo:** `man dnf` (`history`, `module`, `group`, `provides`, `--disablerepo/--enablerepo`), `man dnf.conf` (opciones de repo: `baseurl`, `gpgcheck`, `gpgkey`, `skip_if_unavailable`, `cost`, `priority`), `man dnf-config-manager`. Practicar el ciclo EPEL completo (CRB → epel-release → llave → htop → set-disabled) y el repo desde ISO, incluyendo el error cuando la ISO está desmontada y el repo habilitado.
  4. **subscription-manager repos:** `man subscription-manager` (`repos --list`, `--enable`, `release --set`). Entender que `redhat.repo` se regenera y que `config-manager` no es la vía para esos repos.
  5. **Procesos en RHEL:** `man ps` ("PROCESS STATE CODES"), `man 7 signal`, `man top` (teclas), `man tuned-adm`. Como perfil de seguridad, preparar la explicación del ticket 1 con `/proc/PID/exe`, PPID 2 y corchetes para hilos del kernel, y `rpm -qf`/`rpm -Va` como comprobación de integridad.

- **Errores frecuentes de los participantes y cómo resolverlos:**

| Síntoma | Causa | Solución |
|---|---|---|
| `dnf`: "This system is not registered with an entitlement server" / "no enabled repositories" | VM sin registrar (snapshot antiguo, OVA) | `sudo subscription-manager register`; comprobar `dnf repolist` |
| `pstree: command not found` / `killall: command not found` | Falta `psmisc` | `sudo dnf install -y psmisc` |
| `dnf download`: "No such command" / `config-manager` no existe | Falta `dnf-plugins-core` | `sudo dnf install -y dnf-plugins-core` |
| Los dos `yes` con `nice` distinto consumen lo mismo | Sin competencia (2 vCPU, 2 procesos) | Usar `taskset -c 0` en ambos (Lab 1.2 paso 7) |
| `renice: Permission denied` | Usuario normal intentando bajar el valor `nice` | Solo root puede mejorar prioridad: `sudo renice` |
| `kill -9` "no funciona" | Proceso en estado `D` (E/S) o es un zombie | Esperar E/S / matar al padre; `ps -o stat` |
| Ctrl+Z "cerró" el programa y la terminal quedó rara | Job suspendido | `jobs`; `fg` y salir bien |
| `systemctl start httpd` → "Interactive authentication required" | Falta `sudo` (polkit no puede preguntar por SSH) | `sudo systemctl ...` |
| `Failed to start monitor.service: Unit monitor.service not found` | Falta `daemon-reload` o el archivo tiene otro nombre/ruta | `sudo systemctl daemon-reload`; `ls /etc/systemd/system/` |
| Warning "unit file changed on disk. Run 'systemctl daemon-reload'" | Editó el unit file sin recargar | `sudo systemctl daemon-reload` |
| `monitor.service` falla con `status=203/EXEC` | Script sin `+x`, sin shebang, o con contexto SELinux `user_home_t` (creado en `/home` y movido) | `chmod 755`; primera línea `#!/bin/bash`; `sudo restorecon -v /usr/local/bin/monitor.sh` |
| `Unit httpd.service is masked` | Se olvidó `unmask` | `sudo systemctl unmask httpd` y luego `enable --now` |
| `Failed to mask unit: File /etc/systemd/system/monitor.service already exists` | Intenta enmascarar una unidad propia de `/etc` | Comportamiento esperado; explicarlo (reto) |
| `journalctl -b -1`: "no persistent journal was found" | Journal volátil | Lab 3.2; solo se ve tras reiniciar |
| `journalctl` muestra "not seeing messages from other users" | Usuario fuera de `wheel`/`adm`/`systemd-journal` | `sudo journalctl` (student en `wheel` no debería verlo) |
| El mensaje de `logger -p local0` no aparece en `/var/log/monitor.log` | rsyslog no reiniciado, o error de sintaxis en el `.conf` | `sudo rsyslogd -N1`; `sudo systemctl restart rsyslog`; `systemctl status rsyslog` |
| El mensaje aparece en `messages` **y** en `monitor.log` | Falta `& stop` | Añadirlo debajo de la regla y reiniciar rsyslog |
| `dnf install htop`: "No match for argument: htop" | EPEL no instalado o deshabilitado | Lab 4.2; `dnf repolist --all \| grep epel` |
| `dnf`: "Public key for ... is not installed" / "GPG check FAILED" | Llave del repo no importada o `gpgkey=` mal | `rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-9`; revisar la ruta en el `.repo` |
| `dnf`: "Failed to download metadata for repo 'dvd-baseos'" | ISO desmontada/desconectada con el repo habilitado | Montar de nuevo o `config-manager --set-disabled dvd-baseos dvd-appstream` |
| `mount: /mnt/dvd: no medium found on /dev/sr0` o no existe `sr0` | La ISO no quedó conectada en el hipervisor | Repetir el paso 1 del Lab 4.3; `lsblk`; en UTM `lsblk -f \| grep iso9660` |
| `rpm -ivh` → "Failed dependencies" | Es el comportamiento de rpm | `sudo dnf install ./archivo.rpm` |
| `dnf history undo N` falla: "No package ... available" | Se intenta deshacer una **eliminación** de un paquete que ya no está en ningún repo habilitado | Habilitar el repo (`--enablerepo`) o instalar a mano |
| Al salir de `top`/`htop`, la terminal muestra caracteres extraños | Salió con Ctrl+C a mitad de redibujado | `reset` |

- **Diferencias VirtualBox (x86_64) vs UTM (aarch64):**
  - Los ids de repositorio llevan la arquitectura: `rhel-9-for-x86_64-baseos-rpms` vs `rhel-9-for-aarch64-baseos-rpms`; por eso el material usa `$(arch)` al habilitar CRB. EPEL existe para aarch64 con los mismos paquetes del día (`htop`, `fail2ban`).
  - Los archivos `.rpm` terminan en `.aarch64.rpm`: usar comodines (`tree-*.rpm`) en el Lab 4.1.
  - Conectar la ISO en caliente: VirtualBox por *Devices → Optical Drives*; UTM por el icono de unidades de la ventana. En UTM el lector normalmente es `/dev/sr0` (interfaz USB); el disco principal es `/dev/vda`. ⚠️ Verificar en la VM antes de la clase: si el lector aparece con otro nombre, ajustar `mount` y el `baseurl` del Lab 4.3. El ISO aarch64 no tiene `isolinux/` y la etiqueta es `RHEL-9-4-0-BaseOS-aarch64`.
  - Particionado: en VirtualBox (BIOS) el `lsblk` del Lab 4.3 muestra `sda1` (`/boot`) + `sda2` (LVM); en UTM (UEFI) hay además una partición `/boot/efi` (`vda1`), así que la numeración se corre: `vda1` `/boot/efi`, `vda2` `/boot`, `vda3` LVM.
  - `/proc/cpuinfo` en aarch64 no muestra "model name" (muestra `Features`, `BogoMIPS`): usar `lscpu` para hablar de CPU. `nproc` y `uptime` funcionan igual.
  - Nombres de kernel: `5.14.0-427.x.el9_4.aarch64`. Todo lo demás (systemd, journald, rsyslog, dnf) es idéntico.

- **Preguntas probables y respuesta corta:**
  - *¿`kill -9` directamente no es más rápido?* Más rápido, pero el proceso no cierra archivos ni conexiones ni avisa a sus hijos: bases de datos corruptas y archivos `.lock` huérfanos vienen de ahí. `TERM`, esperar, y solo entonces `KILL`.
  - *¿Por qué el load average es alto si la CPU está ociosa?* Porque cuenta procesos en `D` (esperando disco/red), no solo los que usan CPU. `vmstat` (columna `b`, `wa`) e `iostat` (sysstat) lo confirman.
  - *¿`enable` arranca el servicio?* No. `enable` es "al próximo arranque"; `start` es "ahora". `enable --now` hace ambas cosas.
  - *¿Cuándo usar `mask`?* Cuando un servicio no debe correr bajo ningún concepto (reemplazado por otro, conflicto, política de seguridad), y no basta con `disable` porque otra unidad podría arrancarlo como dependencia.
  - *¿`yum` o `dnf`?* Lo mismo en RHEL 8 y 9 (`yum` es un enlace). En documentación vieja se ve `yum`; escribir `dnf`.
  - *¿Por qué RHEL no trae `htop`?* Red Hat solo incluye lo que se compromete a soportar 10 años. Lo demás vive en EPEL, mantenido por la comunidad.
  - *¿EPEL es seguro en producción?* Es confiable (firmado, proyecto Fedora) pero no soportado por Red Hat. Política razonable: deshabilitado por defecto, paquetes concretos con `--enablerepo`, y revisar `dnf history` tras cada cambio.
  - *¿Cómo aplico solo parches de seguridad?* `sudo dnf update --security` (`dnf updateinfo list security` para verlos): funciona con los repos de Red Hat porque publican metadatos de erratas.
  - *¿Cómo evito que se actualice un paquete?* `exclude=kernel*` en `/etc/dnf/dnf.conf` o en el `.repo`, o el plugin `versionlock` (`python3-dnf-plugin-versionlock`).
  - *¿Hay que reiniciar tras `dnf update`?* Sí si cambió el kernel, glibc o systemd; `sudo dnf needs-restarting -r` lo dice, y `needs-restarting` sin `-r` lista los servicios que cargan librerías viejas.
  - *¿`journalctl` o `/var/log/messages`?* Los dos tienen lo mismo; `journalctl` filtra mejor (unidad, tiempo, prioridad) y `messages` es texto plano que sobrevive, se rota y se puede enviar a un servidor central (rsyslog remoto, Día 10).
  - *¿Cuánto espacio ocupa el journal?* `--disk-usage`; límite por `SystemMaxUse` (default 10 % del disco, máximo 4 GB); `sudo journalctl --vacuum-size=200M` recorta.
  - *¿Cómo hago que mi servicio arranque después de la base de datos?* `After=postgresql.service` (orden) y `Requires=` o `Wants=` (dependencia) en `[Unit]`.
  - *¿Y si el servidor no tiene internet ni lector de DVD?* Copiar la ISO al disco (`scp`) y montarla con `-o loop`; o un espejo interno con `reposync` + `httpd`; o Satellite.
  - *¿Qué es Cockpit?* La consola web (puerto 9090, `cockpit.socket`) que hace por interfaz lo mismo que `systemctl`, `journalctl` y `dnf`. Se ve el Día 07.

- **Relación con el examen RHCSA (EX200):** este día cubre completo el objetivo **"Manage running systems"** en su parte de procesos: *identify CPU/memory intensive processes and kill processes* (`top`, `ps --sort`, `kill`/`pkill`), *adjust process scheduling* (`nice`/`renice`), *manage tuning profiles* (`tuned-adm`), *locate and interpret system log files and journals* (`/var/log`, `journalctl`), **preserve system journals** (Lab 3.2), *start, stop, and check the status of network services* (`systemctl`). De **"Deploy, configure, and maintain systems"**: *start and stop services and configure services to start automatically at boot* (`enable --now`), *configure systems to boot into a specific target automatically* (`set-default`), *configure time service clients* (`timedatectl`, `chronyc`), **install and update software packages from Red Hat Network, a remote repository, or from the local file system** (Labs 4.1–4.3: los tres orígenes), *modify the system bootloader* queda para el Día 10. Formato típico del examen: "Configure el sistema para que arranque en `multi-user.target`", "Cree un repositorio con `baseurl=http://servidor/BaseOS` y `gpgcheck=0`, e instale el paquete X", "Haga que el journal sea persistente", "El proceso que más CPU consume debe ser terminado", "Asegúrese de que `httpd` arranque en cada reinicio". Trampas habituales: olvidar `daemon-reload`, confundir `enable` con `start`, olvidar `enabled=1` en el `.repo`, y no hacer `mkdir /var/log/journal`.

## Tarea y preparación para el día siguiente

1. **Reiniciar y comprobar persistencia** (5 min): `sudo reboot`; al volver, `systemctl is-active monitor httpd` debe responder `active` dos veces, `journalctl --list-boots` debe listar dos arranques (`-1` y `0`) y `sudo journalctl -b -1 -n 3` debe mostrar las últimas líneas del arranque anterior. `sudo tail -2 /var/log/monitor.log` con horas posteriores al reinicio. Si algo no está, es el primer ticket de la tarea.
2. **Snapshot `dia04-fin`** con la VM apagada (`sudo poweroff`). `httpd`, `monitor.service`, EPEL, CRB, `tmux`, `sysstat` y el repo del DVD (deshabilitado) quedan para los días siguientes; el reto puede quedar resuelto o no.
3. **Práctica de 20 minutos** (sin mirar el material; luego comparar):
   - Lanzar `sleep 900 &` con `nice -n 15`, comprobar su `NI` con `ps -o pid,ni,cmd -C sleep`, subirlo a 19, intentar bajarlo a 0 sin `sudo`, y matarlo con `pkill`.
   - Crear `hora.service` (`Type=oneshot`, `ExecStart=/usr/bin/date`) en `/etc/systemd/system/`, arrancarlo, ver `Active: inactive (dead)` y su salida en `journalctl -u hora`.
   - Con `journalctl`, listar los errores (`-p err`) del arranque actual y los mensajes de `sshd` de hoy; contar cuántas veces se usó `sudo` hoy (`-t sudo --since today | wc -l`).
   - Averiguar con una sola consulta `rpm` qué paquete instaló `/usr/sbin/sshd` y qué archivos de configuración tiene; con `dnf`, qué paquete daría el comando `nmap`.
4. **Lectura corta:** `man systemd.service` (solo `Type=` y `Restart=`), `man journalctl` (sección EXAMPLES) y `man 5 dnf.conf` (sección "Repo Options": `baseurl`, `enabled`, `gpgcheck`, `gpgkey`).
5. **Para el Día 05** (redes, NetworkManager, SSH): no hace falta preparar nada nuevo en la VM. Verificar que la suscripción sigue activa (`sudo dnf repolist`) porque se instalan `bind-utils`, `tcpdump`, `traceroute`, `rsync` y `mtr`; tener el hipervisor a mano porque se añadirá un segundo adaptador de red (Host-only) durante la clase; y comprobar que `ssh -p 2222 student@localhost` funciona desde el equipo propio.
