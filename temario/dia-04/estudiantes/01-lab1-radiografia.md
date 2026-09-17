# Lab 1 — Radiografía de procesos

Vamos a leer qué corre en la VM, quién lo lanzó y en qué estado está. Hoy en este lab **no se mata nada**: solo se mira.

---

## Parte 1 — Mi propia shell y sus ancestros

**¿Cuál es el PID de la terminal en la que estoy escribiendo, y quién es su padre?**

```bash
ps
```

Anotar el PID de la línea `bash`. Con **ese número** (acá es `2351`):

```bash
ps -o pid,ppid,user,stat,cmd -p 2351
pstree -ps 2351
```

Foto.

**Comprobar:**
```
    PID TTY          TIME CMD
   2351 pts/0    00:00:00 bash
   2402 pts/0    00:00:00 ps
    PID    PPID USER     STAT CMD
   2351    2350 student  Ss   -bash
systemd(1)───sshd(890)───sshd(2345)───sshd(2350)───bash(2351)───pstree(2402)
```
La cadena arranca en `systemd(1)`. `Ss` = dormida y líder de sesión.

---

## Parte 2 — Todos los procesos, en las dos formas clásicas

**¿Cuántos procesos corren en un servidor que "no está haciendo nada"?**

```bash
ps aux | head -5
ps -ef | head -5
ps aux | wc -l
```

Foto.

**Comprobar:**
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.4 107324 17024 ?        Ss   09:02   0:02 /usr/lib/systemd/systemd ...
root           2  0.0  0.0      0     0 ?        S    09:02   0:00 [kthreadd]
...
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 09:02 ?        00:00:02 /usr/lib/systemd/systemd ...
root           2       0  0 09:02 ?        00:00:00 [kthreadd]
...
130
```
`aux` muestra `%CPU`, `%MEM` y `STAT`; `-ef` muestra el `PPID`. Entre 120 y 150 procesos es lo normal.

---

## Parte 3 — Los que más consumen

**¿Qué comando corro primero cuando llega el ticket "el servidor está lento"?**

```bash
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu | head -8
ps -eo pid,user,%mem,cmd --sort=-%mem | head -5
```

Foto.

**Comprobar:**
```
    PID    PPID USER     %CPU %MEM STAT CMD
      1       0 root      0.1  0.4 Ss   /usr/lib/systemd/systemd ...
    890       1 root      0.0  0.2 Ss   sshd: /usr/sbin/sshd -D ...
    712       1 root      0.0  0.8 Ssl  /usr/lib/systemd/systemd-journald
...
```
Ahora todo está cerca de `0.0`: la VM está tranquila.

---

## Parte 4 — El árbol y los `sshd`

**¿Por qué hay tres `sshd` si yo abrí una sola conexión?**

```bash
pstree -p | head -15
pgrep -a sshd
```

Foto.

**Comprobar:**
```
systemd(1)─┬─NetworkManager(780)─┬─{NetworkManager}(782)
           │                     └─{NetworkManager}(784)
           ├─agetty(905)
           ├─auditd(690)───{auditd}(691)
           ├─chronyd(760)
           ├─crond(900)
           ...
890 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
2345 sshd: student [priv]
2350 sshd: student@pts/0
```
Las llaves `{}` son hilos del mismo proceso. El primer `sshd` escucha, el segundo es la conexión como root, el tercero es tu sesión como `student`.

---

## Parte 5 — `top`

```bash
top
```

En este orden: `1` (una línea por núcleo) · `M` (por memoria) · `P` (por CPU) · `u`, escribir `student`, Enter · `c` · `q` para salir.

Foto de la pantalla antes de salir.

**Comprobar:**
```
top - 09:40:12 up 38 min,  1 user,  load average: 0.05, 0.08, 0.04
Tasks: 130 total,   1 running, 129 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.3 us,  0.2 sy, ...  99.5 id, ...
%Cpu1  :  0.0 us,  0.0 sy, ... 100.0 id, ...
MiB Mem :   3713.9 total,   3010.2 free,    353.4 used,    350.3 buff/cache
MiB Swap:   4027.0 total,   4027.0 free,      0.0 used.   3129.5 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2402 student   20   0   10952   4224   3456 R   0.3   0.1   0:00.02 top
```
Dos líneas `%Cpu`: dos núcleos. `id` = ocioso. `1 running` es el propio `top`.

---

## Parte 6 — Mirar adentro de `/proc`

**¿Cómo sé qué programa está corriendo de verdad un proceso, sin confiar en el nombre?**

Con el PID de tu `bash` de la Parte 1:

```bash
head -8 /proc/2351/status
ls -l /proc/2351/exe /proc/2351/cwd
sudo ls -l /proc/1/exe
sudo ls -l /proc/2/exe
```

Foto.

**Comprobar:**
```
Name:	bash
Umask:	0002
State:	S (sleeping)
Tgid:	2351
Ngid:	0
Pid:	2351
PPid:	2350
TracerPid:	0
lrwxrwxrwx. 1 student student 0 ... /proc/2351/cwd -> /home/student
lrwxrwxrwx. 1 student student 0 ... /proc/2351/exe -> /usr/bin/bash
lrwxrwxrwx. 1 root root 0 ... /proc/1/exe -> /usr/lib/systemd/systemd
ls: cannot read symbolic link '/proc/2/exe': No such file or directory
```
`exe` apunta al programa **real**. PID 2 es un hilo del kernel: no tiene programa en disco.
