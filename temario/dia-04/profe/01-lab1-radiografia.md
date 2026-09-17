# Lab — Radiografía de procesos (comandos)

Guiado: vos tipeás, ellos tipean, todos leen la salida. No se mata nada. Si `pstree` dice `command not found`: `sudo dnf install -y psmisc`.

## Parte 1 — Mi propia shell y sus ancestros
```bash
ps
```
Leer el PID de la línea `bash`. Cada uno usa **su** número (acá `2351`):
```bash
ps -o pid,ppid,user,stat,cmd -p 2351
pstree -ps 2351
```
Qué decir: "el PPID de tu `bash` es un `sshd`: la conexión por la que entraste. Cerrás SSH, muere el padre, muere la shell."

## Parte 2 — Todos los procesos
```bash
ps aux | head -5
ps -ef | head -5
ps aux | wc -l
```

## Parte 3 — Los que más consumen
```bash
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu | head -8
ps -eo pid,user,%mem,cmd --sort=-%mem | head -5
```
Qué decir: "este es el primer comando que corro cuando alguien dice 'el servidor está lento'."

## Parte 4 — El árbol y los `sshd`
```bash
pstree -p | head -15
pgrep -a sshd
```
Respuesta a la pregunta: el primero (`-D [listener]`) escucha en el puerto; el segundo (`student [priv]`) es la conexión, corre como root; el tercero (`student@pts/0`) es la sesión ya como `student`. Separación de privilegios: si comprometen la sesión, no tienen root.

## Parte 5 — `top`
```bash
top
```
Teclas, en orden: `1` · `M` · `P` · `u` `student` Enter · `c` · `q`.
Si alguien sale con Ctrl+C y la terminal queda rara: `reset`.

## Parte 6 — Mirar adentro de `/proc`
Con el PID de la Parte 1:
```bash
head -8 /proc/2351/status
ls -l /proc/2351/exe /proc/2351/cwd
sudo ls -l /proc/1/exe
sudo ls -l /proc/2/exe
```
Qué decir: "el `exe` de PID 2 no existe: es un hilo del kernel. Todo lo que está entre corchetes es así. Un proceso que se llame parecido pero tenga `exe`, no es del kernel."
