# Día 02 — La terminal a fondo

> Al terminar el día, el participante se mueve con soltura por el árbol de directorios de RHEL 9, crea y organiza archivos y enlaces, analiza un log con `grep`/`sort`/`uniq`/`cut`, encuentra cualquier archivo con `find`, respalda y restaura con `tar` y edita un archivo de configuración con `vim` sin ayuda.

**Ficha técnica cubierta:** RH124 M1: Shell y terminal, Navegación del sistema (a fondo) · RH124 M2: Crear/copiar/mover archivos, Compresión y empaquetado, Búsquedas avanzadas.

**Requisitos previos:**
- VM `rhel01` instalada el Día 1 (RHEL 9.x Server sin GUI), snapshot `dia01-fin` tomado.
- Usuario `student` con `sudo` funcionando y contraseña de root conocida.
- Acceso por SSH desde el equipo propio: `ssh -p 2222 student@localhost`.
- Sistema registrado con `subscription-manager` (Día 1). Se verifica en los primeros 10 minutos con `sudo dnf repolist`: hoy se instalan paquetes (`tree`, `zip`, `unzip`, `bzip2`, `mlocate`; `vim-enhanced` y `bash-completion` ya se instalaron el Día 1 y se reinstalan sin daño si faltan).
- No hacen falta discos ni adaptadores de red adicionales.

---

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:10 | 10 | Repaso y conexión | Todos conectados por SSH; `dnf repolist` OK; tres preguntas del Día 1; instalación de los paquetes del día |
| 0:10–0:30 | 20 | Bloque 1 · Conceptos | Anatomía de un comando, prompt, rutas, variables de entorno, alias, historial, atajos; jerarquía FHS de RHEL 9 |
| 0:30–0:50 | 20 | Lab 1.1 | Recorrido guiado por el árbol y por la shell (variables, alias, historial, Tab, man) |
| 0:50–1:05 | 15 | Bloque 2 · Conceptos | `ls -l` columna por columna, tipos de archivo, `cp -a` vs `cp -r`, `rm` y sus peligros, enlaces duros y simbólicos, globbing |
| 1:05–1:30 | 25 | Lab 2.1 | Construir `~/empresa` (estructura, copias, movimientos, enlaces, `stat`, `file`, globbing, `tree`) |
| 1:30–1:45 | 15 | Bloque 3 · Conceptos | Herramientas de texto, redirección, pipes, `tee`, `xargs`, `grep` y expresiones regulares básicas |
| 1:45–2:00 | 15 | Descanso | |
| 2:00–2:30 | 30 | Lab 3.1 | Generar un log de 300 líneas y analizarlo con `grep`/`sort`/`uniq`/`cut`/`wc` y redirección; `tail -f`; logs reales de `/var/log` |
| 2:30–2:40 | 10 | Bloque 4 · Conceptos | `find` (criterios y acciones), `locate`/`updatedb`, `which`/`whereis`/`type` |
| 2:40–2:55 | 15 | Lab 4.1 | `find` sobre `/etc` y `/var/log` con `-exec`; instalar y usar `locate` |
| 2:55–3:15 | 20 | Bloque 5 · Conceptos + Lab 5.1 | `tar`, `gzip`/`bzip2`/`xz`, `zip`; respaldar `~/empresa` y `/etc`, listar, restaurar y verificar |
| 3:15–3:30 | 15 | Bloque 6 · Conceptos + Lab 6.1 | `vim` (modos, guardar, salir, buscar, `:%s`) y `nano` como alternativa |
| 3:30–3:50 | 20 | Reto individual | Ticket: analizar un log de 200 líneas, empaquetar con fecha y crear enlace `~/ultimo-backup` |
| 3:50–4:00 | 10 | Cierre | Resumen, cheatsheet, snapshot `dia02-fin`, tarea |

Total: 240 min.

---

## Prioridad si falta tiempo

**Imprescindible** (el participante debe salir sabiéndolo y habiéndolo practicado):
- Rutas absolutas y relativas, `.` `..` `~` `-`, `pwd`, `cd`, `ls -la`, `ls -lh`, lectura de las columnas de `ls -l`.
- `mkdir -p`, `touch`, `cp -r`, `mv`, `rm -r` (y por qué `rm -rf` es peligroso), `rmdir`.
- `cat`, `less`, `head`, `tail -f`, `wc -l`.
- `grep` con `-i -n -v -c -E`, y `grep -E "ERROR|WARN"`.
- Redirección `>` `>>` `2>` y pipes con `sort | uniq -c`; por qué `sudo echo > /etc/x` falla y `| sudo tee` funciona.
- `find /ruta -name`, `-type`, `-size`, `-mtime`.
- `tar -czf`, `tar -tzf`, `tar -xzf ... -C`.
- `vim`: `i`, `Esc`, `:wq`, `:q!`, `dd`, `u`.

**Importante:**
- Enlaces duros vs simbólicos (`ln`, `ln -s`, cómo se ven en `ls -l`), `stat`, `file`.
- Globbing (`* ? []`) y expansión de llaves (`{1..5}`).
- `cut -d -f`, `sort -n -r -k -u`, `uniq -c`, `tee`, `xargs`, `2>&1`, `/dev/null`.
- `find -mmin -user -perm -empty -newer`, `-exec {} \;` vs `-exec {} +`, `-delete`.
- `gzip`/`bzip2`/`xz` con comparación de tamaños; respaldo y restauración de `/etc`.
- Alias persistentes en `~/.bashrc`, `history`, `!!`, `Ctrl+R`, atajos `Ctrl+A/E/U/K/W`.
- `vim`: `/buscar`, `n`, `:set nu`, `yy`, `p`, `:%s/viejo/nuevo/g`.

**Si sobra tiempo** (demo del instructor o tarea):
- `locate`/`updatedb`, `whereis`, `type`.
- `zip`/`unzip`, `tar -cjf`/`-cJf`, `tar -a`.
- `diff`, `nl`, `tr`, truco de `tail -f` con subshell, `less` avanzado (`&`, `F`).
- `PS1` personalizado, `HISTTIMEFORMAT`, `set -o noclobber`.
- Exploración de `/proc` y `/sys`; `nano`; `vimtutor`.

---

## Bloque 0 — Repaso y conexión (10 min)

1. Todos conectan desde su propio equipo:

```bash
ssh -p 2222 student@localhost
hostname && cat /etc/redhat-release && uname -m
```

Salida esperada:
```
rhel01
Red Hat Enterprise Linux release 9.x (Plow)
x86_64
```
Qué observar: la versión menor (9.4, 9.5, 9.6...) depende de la ISO usada el Día 1; todos deben ver `9.x`. En la VM del instructor la arquitectura es `aarch64`. Todo lo demás del día es idéntico.

2. Confirmar que `dnf` tiene repositorios (registro del Día 1):

```bash
sudo dnf repolist
```

Salida esperada:
```
repo id                                   repo name
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
```
Si aparece `This system is not registered` o la lista está vacía: `sudo subscription-manager register` con la cuenta de developers.redhat.com (ver Día 1). Con Simple Content Access no hace falta `attach`.

3. Instalar de una vez las herramientas del día (evita interrumpir los labs):

```bash
sudo dnf install -y tree vim-enhanced bash-completion zip unzip bzip2 nano
```

Salida esperada (resumida):
```
...
Installed:
  tree-1.8.0-10.el9.x86_64   vim-enhanced-2:8.2.2637-20.el9_1.x86_64   zip-3.0-35.el9.x86_64 ...
Complete!
```
Qué observar: si algún paquete ya estaba (`vim-enhanced`, `bash-completion` y `nano` normalmente ya están), dnf dice `Package ... is already installed` y sigue. `mlocate` se instala en el Bloque 4. Las versiones exactas (`el9_1`, `el9_4`...) varían con la versión menor de RHEL.

4. Tres preguntas rápidas del Día 1 (respuesta oral): ¿qué diferencia hay entre kernel y distribución? ¿por qué usamos `sudo` en lugar de trabajar como root? ¿por qué entramos por el puerto 2222 y no por el 22?

---

## Bloque 1 — La shell y el árbol de RHEL 9

### Conceptos (20 min)

**Qué es la shell.** Bash es el programa que lee lo que el participante escribe, lo interpreta y ejecuta programas. Analogía: es el "intérprete" entre la persona y el kernel. El *prompt* es la señal de que la shell está lista:

```
[student@rhel01 ~]$
```

`student` = usuario, `rhel01` = equipo, `~` = directorio actual (el home), `$` = usuario normal (`#` sería root). Esa forma la define la variable `PS1`.

**Anatomía de un comando.** `comando [opciones] [argumentos]`:

```
ls   -l -a -h   /etc
ls   -lah       /etc        # opciones cortas se pueden combinar
ls   --all -l   /etc        # opciones largas (--) no se combinan
```
- Opciones cortas: un guion, una letra, combinables (`-lah`).
- Opciones largas: dos guiones, una palabra (`--human-readable`), más legibles en scripts.
- Argumentos: sobre qué actúa el comando (archivos, directorios, textos).
- `--` marca el fin de las opciones: `rm -- -archivo` borra un archivo llamado `-archivo`.
- La documentación oficial es `man comando`; `comando --help` da el resumen; `man -k palabra` (o `apropos`) busca en las descripciones. Hay que decirlo en clase: si `man -k` responde `nothing appropriate`, la base de datos de man aún no se creó en esa VM nueva: `sudo mandb`.

**Rutas.**
- Absoluta: empieza en `/` y no depende de dónde se está (`/var/log/messages`).
- Relativa: se interpreta desde el directorio actual (`logs/servidor.log`).
- `.` = directorio actual, `..` = directorio padre, `~` = home del usuario (`/home/student`), `~root` = `/root`, `-` en `cd -` = el directorio anterior.
- Regla práctica para el administrador: en scripts y en `cron`, siempre rutas absolutas.

**Variables de entorno.** La shell guarda su configuración en variables. Se leen con `$NOMBRE`:
- `HOME` (home del usuario), `USER`, `SHELL` (`/bin/bash`), `PWD`, `OLDPWD`, `HOSTNAME`.
- `PATH`: lista de directorios, separados por `:`, donde la shell busca los programas. Por eso `ls` funciona sin escribir `/usr/bin/ls`, y por eso un script en el directorio actual se ejecuta como `./script.sh` (el `.` no está en el PATH por seguridad).
- `PS1`: el formato del prompt. `LANG`: idioma y ordenación (afecta a `sort` y a los mensajes).
- `env` o `printenv` muestran las variables exportadas; `set` muestra también las locales y funciones.
- Una variable creada en la shell (`MIVAR=hola`) solo existe en esa shell; con `export MIVAR` la heredan los programas que se lancen desde ella. Esto será clave en scripts (Día 7).
- `$?` es el código de salida del último comando: `0` = éxito, otro valor = error. Los scripts y `cron` viven de este valor.

**Alias.** Un apodo para un comando largo: `alias lh='ls -lh'`. Solo dura la sesión; para hacerlo permanente se agrega a `~/.bashrc`. RHEL ya trae varios (`ll`, `ls --color=auto`, `grep --color=auto`), definidos en `/etc/profile.d/*.sh` (`colorls.sh`, `colorgrep.sh`, `which2.sh`) y en `/etc/bashrc`. ⚠️ Verificar en la VM antes de la clase: la lista exacta cambia de una versión menor a otra (por ejemplo, `egrep` puede aparecer como `egrep --color=auto` o como `grep -E --color=auto`); basta con ejecutar `alias` y comparar. `type comando` dice si algo es alias, builtin o programa.

**Historial.** Bash guarda lo escrito en `~/.bash_history` (1000 líneas por defecto en RHEL, `HISTSIZE`).
- `history` lista; `!!` repite el último comando (`sudo !!` es el uso estrella); `!n` repite el número n; `!$` es el último argumento del comando anterior.
- `Ctrl+R` busca hacia atrás escribiendo un fragmento; `Enter` ejecuta, `Esc` deja el comando en la línea para editarlo.
- Cuidado con `!` dentro de comillas dobles: `echo "Hola!"` falla con `event not found`. Usar comillas simples.

**Atajos de edición de línea** (readline, funcionan también en `mysql`, `python`, etc.):

| Atajo | Efecto |
|---|---|
| `Tab` / `Tab Tab` | Completar comando, ruta o nombre de servicio / listar opciones |
| `Ctrl+A` / `Ctrl+E` | Inicio / fin de línea |
| `Ctrl+U` / `Ctrl+K` | Cortar desde el cursor hasta el inicio / hasta el final |
| `Ctrl+W` | Cortar hacia atrás hasta el espacio anterior: en una ruta se lleva la ruta entera, no solo el último tramo. `Alt+Backspace` corta menos: solo hasta el separador anterior (`/`, `-`, `.`) |
| `Ctrl+Y` | Pegar lo cortado |
| `Ctrl+L` | Limpiar pantalla (igual que `clear`) |
| `Ctrl+C` | Cancelar el comando en curso |
| `Ctrl+D` | Fin de entrada; en un prompt vacío cierra la shell (y la sesión SSH) |
| `Ctrl+Z` | Suspende el proceso (se recupera con `fg`); si alguien "perdió" vim, probablemente hizo esto |
| `Ctrl+S` / `Ctrl+Q` | Congela / descongela la terminal. "Se me trabó la terminal" casi siempre es `Ctrl+S` |

**Jerarquía del sistema de archivos (FHS) en RHEL 9.** Todo cuelga de `/`. Analogía: un edificio con pisos con función fija; en Windows cada programa se instala en su carpeta, en Linux cada *tipo* de archivo tiene su sitio.

| Directorio | Contenido | Nota para RHEL 9 |
|---|---|---|
| `/` | Raíz de todo | |
| `/etc` | Configuración del sistema y servicios (texto plano) | Es lo que se respalda antes de tocar algo |
| `/var` | Datos variables: logs, colas, cachés, bases de datos | |
| `/var/log` | Logs del sistema (`messages`, `secure`, `httpd/`) | `journalctl` complementa (Día 4) |
| `/usr` | Programas y librerías instalados por el sistema (`/usr/bin`, `/usr/sbin`, `/usr/lib64`, `/usr/share/doc`) | |
| `/bin`, `/sbin`, `/lib`, `/lib64` | Enlaces simbólicos a `/usr/bin`, `/usr/sbin`, `/usr/lib`, `/usr/lib64` | Desde RHEL 7 ("UsrMove"). `/bin/bash` y `/usr/bin/bash` son el mismo archivo |
| `/home` | Homes de los usuarios normales | `/home/student` |
| `/root` | Home de root | No está en `/home`; permisos `550` |
| `/tmp` | Temporales; cualquiera escribe | systemd-tmpfiles borra lo no usado en 10 días |
| `/opt` | Software de terceros autocontenido | Ej. agentes de monitoreo, Oracle |
| `/srv` | Datos servidos por servicios (web, ftp, compartidos) | Lo usaremos para `/srv/sistemas`, `/srv/soporte` y `/srv/publico` (Día 3) |
| `/dev` | Dispositivos como archivos: discos, terminales, `/dev/null` | `/dev/sda` (VirtualBox) vs `/dev/vda` (UTM) |
| `/proc` | Sistema virtual del kernel: procesos y estado (`/proc/cpuinfo`) | Ocupa 0 bytes en disco |
| `/sys` | Sistema virtual de hardware y drivers | |
| `/boot` | Kernel (`vmlinuz-*`), `initramfs`, GRUB | `/boot/efi` con contenido solo si el arranque es UEFI (UTM sí; VirtualBox solo si se activó EFI al crear la VM); en BIOS el directorio puede existir vacío |
| `/mnt` | Montajes manuales del administrador | |
| `/media` | Montajes de medios extraíbles | Casi vacío en servidores |
| `/run` | Datos de ejecución (PIDs, sockets), en RAM, se vacía al arrancar | `/var/run` es enlace a `/run` |

Referencia: `man 7 hier`.

### Lab 1.1 — Recorrido por la shell y el árbol (20 min)

- **Objetivo:** leer el prompt, dominar rutas, variables, alias e historial, y reconocer cada directorio de primer nivel de RHEL 9.
- **Ritmo:** son 14 pasos cortos; los pasos 1, 3–7, 10 y 11 son el núcleo. Si el grupo va lento, el instructor hace 2, 8, 9, 12 y 13 como demo compartiendo pantalla y deja 14 como lectura en casa.

1. Leer el prompt y su definición.

```bash
echo $PS1
```
Salida esperada:
```
[\u@\h \W]\$
```
Qué observar: `\u` usuario, `\h` host, `\W` último tramo del directorio, `\$` cambia a `#` si es root.

2. Cambiar el prompt de forma temporal y volver.

```bash
PS1='\u@\h:\w\$ '
pwd
PS1='[\u@\h \W]\$ '
```
Salida esperada:
```
student@rhel01:~$ pwd
/home/student
[student@rhel01 ~]$
```
Qué observar: `\w` muestra la ruta completa. El cambio se pierde al cerrar sesión; para hacerlo fijo iría en `~/.bashrc`.

3. Variables de entorno básicas.

```bash
echo "$USER en $HOSTNAME, home=$HOME, shell=$SHELL"
echo $PATH
env | wc -l
env | grep -E "^(HOME|PATH|LANG|SHELL|USER)="
```
Salida esperada:
```
student en rhel01, home=/home/student, shell=/bin/bash
/home/student/.local/bin:/home/student/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
24
SHELL=/bin/bash
USER=student
PATH=/home/student/.local/bin:/home/student/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
LANG=en_US.UTF-8
HOME=/home/student
```
Qué observar: `LANG` puede ser `es_PA.UTF-8` o `es_ES.UTF-8` si se instaló en español. El PATH incluye `~/bin` aunque no exista todavía.

4. Variable local vs exportada.

```bash
MIVAR=hola
echo $MIVAR
bash
echo "Hija: $MIVAR"
exit
export MIVAR
bash
echo "Hija: $MIVAR"
exit
```
Salida esperada:
```
hola
Hija:
exit
Hija: hola
exit
```
Qué observar: al escribir `bash` el prompt no cambia pero se está en una shell hija. Sin `export`, la hija no ve la variable. Decir: "esto es lo que le pasa a un script que no encuentra una variable que sí está en la terminal".

5. Código de salida.

```bash
ls /etc/hostname; echo "codigo=$?"
ls /noexiste; echo "codigo=$?"
```
Salida esperada:
```
/etc/hostname
codigo=0
ls: cannot access '/noexiste': No such file or directory
codigo=2
```

6. Rutas y `cd`.

```bash
cd /var/log
pwd
cd ..
pwd
cd -
cd ~
pwd
cd /etc/../var/./log && pwd
cd
```
Salida esperada:
```
/var/log
/var
/var/log
/home/student
/var/log
```
Qué observar: `cd -` imprime a dónde volvió; `cd` sin argumentos = home; `..` y `.` se pueden encadenar.

7. Primer nivel del árbol.

```bash
ls -l /
```
Salida esperada (resumida):
```
lrwxrwxrwx.   1 root root    7 ... bin -> usr/bin
dr-xr-xr-x.   6 root root 4096 ... boot
drwxr-xr-x.  19 root root 3140 ... dev
drwxr-xr-x.  92 root root 8192 ... etc
drwxr-xr-x.   3 root root   21 ... home
lrwxrwxrwx.   1 root root    7 ... lib -> usr/lib
lrwxrwxrwx.   1 root root    9 ... lib64 -> usr/lib64
drwxr-xr-x.   2 root root    6 ... media
drwxr-xr-x.   2 root root    6 ... mnt
drwxr-xr-x.   2 root root    6 ... opt
dr-xr-xr-x. 260 root root    0 ... proc
dr-xr-x---.   3 root root  ... root
drwxr-xr-x.  29 root root  820 ... run
lrwxrwxrwx.   1 root root    8 ... sbin -> usr/sbin
drwxr-xr-x.   2 root root    6 ... srv
dr-xr-xr-x.  13 root root    0 ... sys
drwxrwxrwt.   8 root root  ... tmp
drwxr-xr-x.  12 root root  144 ... usr
drwxr-xr-x.  20 root root 4096 ... var
```
Qué observar: (a) la `l` inicial y la flecha `->` marcan los enlaces simbólicos `bin`, `sbin`, `lib`, `lib64`; (b) `proc` y `sys` tienen tamaño 0: son virtuales; (c) `tmp` termina en `t` (sticky bit, Día 3); (d) el punto después de los permisos indica que el archivo tiene contexto SELinux (Día 8). Puede aparecer también `afs`, lo crea el paquete `filesystem`; se ignora.

8. Confirmar los enlaces de UsrMove y contar programas.

```bash
ls -ld /bin /sbin /lib /lib64
ls -l /bin/bash /usr/bin/bash
ls /usr/bin | wc -l
ls /etc | wc -l
```
Salida esperada:
```
lrwxrwxrwx. 1 root root 7 ... /bin -> usr/bin
lrwxrwxrwx. 1 root root 7 ... /lib -> usr/lib
lrwxrwxrwx. 1 root root 9 ... /lib64 -> usr/lib64
lrwxrwxrwx. 1 root root 8 ... /sbin -> usr/sbin
-rwxr-xr-x. 1 root root 1390288 ... /bin/bash
-rwxr-xr-x. 1 root root 1390288 ... /usr/bin/bash
1350
175
```
Qué observar: los números varían; lo importante es que `/bin/bash` y `/usr/bin/bash` tienen el mismo tamaño porque son el mismo archivo.

9. Recorrido rápido por los directorios especiales.

```bash
ls /var/log
ls /home; sudo ls -la /root
ls -ld /tmp
ls /dev | head -5
ls /dev/sd* /dev/vd* 2>/dev/null
grep -c processor /proc/cpuinfo; head -3 /proc/meminfo
ls /sys/class/net
ls /boot
ls /run | head -5
ls /opt /srv /mnt /media
```
Salida esperada (resumida):
```
anaconda  audit  btmp  chrony  cron  dnf.log  firewalld  hawkey.log  lastlog  maillog  messages  private  README  secure  spooler  sssd  tuned  wtmp
student
total 28
dr-xr-x---.  3 root root 4096 ... .
drwxr-xr-x. 19 root root  268 ... ..
-rw-r--r--.  1 root root   18 ... .bash_logout
...
drwxrwxrwt. 8 root root 4096 ... /tmp
autofs  block  bsg  btrfs-control  bus
/dev/sda  /dev/sda1  /dev/sda2
2
MemTotal:        3914308 kB
MemFree:         3381784 kB
MemAvailable:    3515960 kB
enp0s3  lo
config-5.14.0-<build>.el9_x.x86_64  efi  grub2  initramfs-5.14.0-<build>.el9_x.x86_64.img  loader  symvers-...  System.map-...  vmlinuz-...
console  cryptsetup  dbus  faillock  initramfs
```
Qué observar: en UTM el disco es `/dev/vda` y la interfaz `enp0s1` (verificar con `nmcli device`). `/opt`, `/srv`, `/mnt` y `/media` están vacíos en una instalación nueva. Si aparece `/var/log/README`, explica que gran parte del log vive en journald (⚠️ Verificar en la VM antes de la clase: el archivo no está en todas las versiones menores).

10. Alias: ver los que trae RHEL, crear uno, hacerlo permanente.

```bash
alias
alias lh='ls -lh'
lh /etc | head -3
type lh; type ls; type cd; which ls
echo "alias lh='ls -lh'" >> ~/.bashrc
source ~/.bashrc
```
Salida esperada (resumida):
```
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias l.='ls -d .* --color=auto'
alias ll='ls -l --color=auto'
alias ls='ls --color=auto'
alias which='(alias; declare -f) | /usr/bin/which --tty-only --read-alias --read-functions --show-tilde --show-dot'
...
total 1.3M
-rw-r--r--.  1 root root   16 ... adjtime
-rw-r--r--.  1 root root 1.5K ... aliases
lh is aliased to `ls -lh'
ls is aliased to `ls --color=auto'
cd is a shell builtin
alias ls='ls --color=auto'
	/usr/bin/ls
```
Qué observar: `ls` ya es un alias en RHEL (por eso colorea). `cd` no es un programa sino parte de bash (builtin). `which` en RHEL también reporta alias.

11. Historial y expansión.

```bash
history | tail -5
cat /etc/shadow
sudo !!
echo 'Hola!'
echo "Hola!"
```
Salida esperada:
```
  495  alias lh='ls -lh'
  496  lh /etc | head -3
  ...
cat: /etc/shadow: Permission denied
sudo cat /etc/shadow
root:$6$...:19800:0:99999:7:::
...
Hola!
-bash: !": event not found
```
Qué observar: `sudo !!` imprime el comando expandido antes de ejecutarlo. El `!` dentro de comillas dobles dispara la expansión de historial; con comillas simples no.

12. `Ctrl+R`: pulsar `Ctrl+R`, escribir `cpuinfo`, ver cómo aparece el comando anterior, `Enter`. Después probar `!$`:

```bash
ls -l /etc/hostname
cat !$
```
Salida esperada:
```
cat /etc/hostname
rhel01
```

13. Atajos y `Tab`. Escribir sin ejecutar `ls -l /etc/NetworkManager/system-connections` y practicar: `Ctrl+A`, `Ctrl+E`, `Ctrl+W` (corta hasta el espacio anterior: se lleva la ruta completa, porque para readline "palabra" es lo que hay entre espacios), `Ctrl+Y` (la devuelve), `Ctrl+U` (borra todo). `Alt+Backspace` corta menos: solo hasta el separador anterior, así que sobre `system-connections` se lleva únicamente `connections` (y además cada terminal lo interpreta a su manera; en Mac hay que activar "Usar Opción como tecla Meta"). Después:

```bash
ls /etc/sys<Tab><Tab>
systemctl sta<Tab>
```
Qué observar: `Tab Tab` lista `sysconfig/ sysctl.conf sysctl.d/ systemd/`; bash también completa subcomandos de `systemctl` gracias al paquete `bash-completion` (instalado el Día 1 y en el Bloque 0; si no completa, abrir una sesión nueva: la completación se carga al iniciar la shell).

14. Documentación en 60 segundos.

```bash
man ls
```
Dentro de `man`: `/-h` busca la opción, `n` siguiente coincidencia, `q` sale.

```bash
ls --help | head -3
man -k "list directory"
whatis ls
man 5 passwd | head -5
```
Salida esperada:
```
Usage: ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).
Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.
ls (1)               - list directory contents
ls (1)               - list directory contents
PASSWD(5)                     File Formats Manual                    PASSWD(5)
```
Qué observar: `man passwd` es el comando (sección 1); `man 5 passwd` es el formato del archivo `/etc/passwd`. Si `man -k` devuelve `nothing appropriate`: `sudo mandb` y repetir.

- **Checkpoint:** pegar en el chat la salida de:

```bash
echo "$USER@$HOSTNAME arch=$(uname -m)"; ls -ld /bin /sbin /lib /lib64; type lh
```

---

## Bloque 2 — Archivos, directorios y enlaces

### Conceptos (15 min)

**Leer `ls -l` columna por columna.**

```
-rw-r--r--. 1 student student 1024 Mar 14 10:22 informe1.txt
│└──┬─────┘│ │    │       │      │        │           └ nombre
│   │      │ │    │       │      │        └ fecha de última modificación
│   │      │ │    │       │      └ tamaño en bytes (-h lo muestra en K/M/G)
│   │      │ │    │       └ grupo propietario
│   │      │ │    └ usuario propietario
│   │      │ └ número de enlaces duros (en directorios: subdirectorios + 2)
│   │      └ punto = tiene contexto SELinux (RHEL)
│   └ permisos rwx para dueño, grupo y otros (Día 3)
└ tipo de archivo
```

**Tipos de archivo** (primer carácter): `-` archivo regular, `d` directorio, `l` enlace simbólico, `c` dispositivo de caracteres (`/dev/null`, `/dev/tty`), `b` dispositivo de bloques (`/dev/sda`), `s` socket, `p` pipe con nombre. En Linux "todo es un archivo": un disco, la terminal y el teclado se manejan con las mismas llamadas.

**Opciones útiles de `ls`:** `-l` largo, `-a` incluye ocultos (los que empiezan por `.`), `-h` tamaños legibles, `-R` recursivo, `-t` ordena por fecha (más reciente primero; `-tr` invierte), `-S` ordena por tamaño, `-d` muestra el directorio en sí y no su contenido, `-i` inodo, `--color=auto` (ya viene por alias: azul directorio, cian enlace, verde ejecutable, rojo comprimido).

**Crear y mover.**
- `mkdir -p a/b/c` crea toda la cadena sin quejarse si existe.
- `touch` crea un archivo vacío o actualiza su fecha (`touch -d "2020-01-15" f` la fija).
- `cp origen destino`: `-r` recursivo (directorios), `-i` pregunta antes de sobrescribir, `-v` cuenta lo que hace, `-p` conserva permisos/fechas, `-a` = archivo: recursivo + conserva todo (permisos, dueño, fechas, enlaces). Regla: para copias de respaldo, `cp -a`; `cp -r` sin más crea archivos "nuevos" con fecha de hoy y dueño quien copia.
- Trampa clásica: `cp -r dir destino` crea `destino/dir` si `destino` existe, pero crea `destino` como copia de `dir` si no existe.
- `mv` mueve o renombra (es lo mismo para Linux): `-i` pregunta, `-v` cuenta, `-n` no sobrescribe. Dentro del mismo filesystem es instantáneo aunque el archivo pese gigas: solo cambia el nombre en el directorio.

**Borrar.** No hay papelera. `rm archivo`, `rm -i` pregunta, `rm -r` recursivo, `rm -f` fuerza sin preguntar y sin error si no existe. `rmdir` solo borra directorios vacíos (es la forma segura).
Hay que decirlo con énfasis: `rm -rf` con una ruta mal escrita, una variable vacía (`rm -rf "$DIR"/*` con `DIR` sin definir se convierte en `rm -rf /*`, que arrasa el sistema; ojo: `--preserve-root` está activo por defecto pero solo protege el argumento `/` exacto, no `/*`) o un espacio de más (`rm -rf ~/empresa /backups`) ha destruido servidores en producción. Hábitos: hacer `ls` de la misma ruta antes de `rm -r`; usar `rm -ri` mientras se aprende; en RHEL el usuario root ya trae `alias rm='rm -i'` por esta razón.

**`file` y `stat`.** `file` identifica el contenido real (no se fía de la extensión): "ASCII text", "ELF 64-bit ... executable", "gzip compressed data", "directory". `stat` muestra todo lo que el sistema sabe del archivo: tamaño, inodo, enlaces, permisos en octal, dueño, contexto SELinux y las tres fechas (Access, Modify, Change).

**Enlaces.** Un archivo es un *inodo* (los datos y metadatos) más uno o más nombres que apuntan a él.
- **Enlace duro** (`ln original nuevo`): otro nombre para el mismo inodo. `ls -li` muestra el mismo número de inodo y el contador de enlaces sube a 2. Si se borra el original, el contenido sigue vivo mientras exista un nombre. Límites: solo dentro del mismo filesystem, no a directorios.
- **Enlace simbólico** (`ln -s destino nombre`): un archivo pequeño que contiene una ruta. `ls -l` lo muestra como `l... nombre -> destino`. Puede cruzar filesystems y apuntar a directorios. Si el destino desaparece, queda "roto" (`ls --color` lo pinta en rojo). Es lo que se usa en la práctica: `/bin -> usr/bin`, versiones de software (`java -> java-17`), "último backup" (`ultimo-backup -> backup-2026-03-14.tar.gz`). Consejo: crear el enlace con la ruta absoluta del destino para que funcione desde cualquier directorio.

**Globbing** (la shell expande los patrones antes de ejecutar el comando):
- `*` cualquier cadena (incluida vacía), `?` exactamente un carácter, `[abc]` uno de esos, `[a-z]`, `[0-9]`, `[!a]` cualquiera menos `a`.
- Expansión de llaves (no es globbing pero se usa junto): `{a,b,c}` genera alternativas, `{1..5}` secuencias. `mkdir -p proyecto/{src,doc,test}` y `touch informe{1..5}.txt` ahorran mucho tiempo.
- `echo patrón` muestra qué expandirá la shell sin hacer nada: útil antes de un `rm`.
- Los archivos ocultos (`.algo`) no entran en `*`.

### Lab 2.1 — Construir `~/empresa` (25 min)

- **Objetivo:** crear la estructura de trabajo del curso y practicar copias, movimientos, borrado, enlaces, `stat`, `file` y globbing.
- **Ritmo:** los pasos 1, 2, 4, 6, 8, 9 y 12 son el núcleo (dejan `~/empresa` como lo necesitan los bloques siguientes). Los pasos 3, 5, 7, 10 y 11 pueden hacerse como demo si falta tiempo.

1. Crear la estructura con expansión de llaves y verla con `tree`.

```bash
mkdir -p ~/empresa/{documentos,clientes,backups,logs}
tree ~/empresa
```
Salida esperada:
```
/home/student/empresa
├── backups
├── clientes
├── documentos
└── logs

4 directories, 0 files
```

2. Crear archivos con contenido y algunos vacíos.

```bash
cd ~/empresa/documentos
for i in {1..5}; do echo "Informe numero $i de PanamaTech" > informe$i.txt; done
touch plan-2026.doc presupuesto.csv
ls
```
Salida esperada:
```
informe1.txt  informe2.txt  informe3.txt  informe4.txt  informe5.txt  plan-2026.doc  presupuesto.csv
```

3. Las variantes de `ls`.

```bash
ls -l
ls -la
ls -lh
ls -lt
ls -lS | head -3
ls -R ~/empresa
ls -d ~/empresa/*/
```
Salida esperada (resumida):
```
total 20
-rw-r--r--. 1 student student 31 ... informe1.txt
...
-rw-r--r--. 1 student student  0 ... plan-2026.doc
-rw-r--r--. 1 student student  0 ... presupuesto.csv
drwxr-xr-x. 2 student student 143 ... .
drwxr-xr-x. 6 student student  71 ... ..
...
/home/student/empresa/backups/  /home/student/empresa/clientes/  /home/student/empresa/documentos/  /home/student/empresa/logs/
```
Qué observar: `-la` muestra `.` y `..`; `-lt` pone primero lo más reciente; `-lS` primero lo más grande; `-d */` lista solo directorios.

4. Copiar: archivo, a otro directorio, recursivo.

```bash
cp informe1.txt copia-informe1.txt
cp -v informe2.txt ../clientes/
cp -r ~/empresa/documentos ~/empresa/backups/
ls -R ~/empresa/backups
```
Salida esperada:
```
'informe2.txt' -> '../clientes/informe2.txt'
/home/student/empresa/backups:
documentos

/home/student/empresa/backups/documentos:
copia-informe1.txt  informe1.txt  informe2.txt  informe3.txt  informe4.txt  informe5.txt  plan-2026.doc  presupuesto.csv
```

5. `cp` normal vs `cp -a`: qué se conserva (con `-r` pasa lo mismo que con `cp` a secas; lo que cambia es `-a`).

```bash
touch -d "2020-01-15 10:00" informe3.txt
cp informe3.txt /tmp/copia-r.txt
cp -a informe3.txt /tmp/copia-a.txt
stat -c '%y  %n' informe3.txt /tmp/copia-r.txt /tmp/copia-a.txt
```
Salida esperada:
```
2020-01-15 10:00:00.000000000 -0500  informe3.txt
2026-09-03 09:31:12.417230198 -0500  /tmp/copia-r.txt
2020-01-15 10:00:00.000000000 -0500  /tmp/copia-a.txt
```
Qué observar: la copia normal tiene fecha de hoy; `-a` conservó la fecha original. En un respaldo, las fechas importan.

6. Mover y renombrar.

```bash
mv informe5.txt ../clientes/cliente-acme.txt
mv -i informe4.txt informe1.txt
mv -v copia-informe1.txt borrador.txt
ls ~/empresa/documentos ~/empresa/clientes
```
Salida esperada:
```
mv: overwrite 'informe1.txt'? n
renamed 'copia-informe1.txt' -> 'borrador.txt'
/home/student/empresa/clientes:
cliente-acme.txt  informe2.txt

/home/student/empresa/documentos:
borrador.txt  informe1.txt  informe2.txt  informe3.txt  informe4.txt  plan-2026.doc  presupuesto.csv
```
Qué observar: responder `n` al `overwrite?`. `mv` sin `-i` habría sobrescrito `informe1.txt` sin avisar.

7. `stat` y `file`.

```bash
stat informe1.txt
file informe1.txt presupuesto.csv /usr/bin/ls ~/empresa /dev/null /bin
```
Salida esperada:
```
  File: informe1.txt
  Size: 31        	Blocks: 8          IO Block: 4096   regular file
Device: fd00h/64768d	Inode: 17512433    Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/ student)   Gid: ( 1000/ student)
Context: unconfined_u:object_r:user_home_t:s0
Access: 2026-09-03 09:28:01.101853420 -0500
Modify: 2026-09-03 09:28:01.101853420 -0500
Change: 2026-09-03 09:28:01.101853420 -0500
 Birth: 2026-09-03 09:28:01.101853420 -0500
informe1.txt:    ASCII text
presupuesto.csv: empty
/usr/bin/ls:     ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, ... stripped
/home/student/empresa: directory
/dev/null:       character special (1/3)
/bin:            symbolic link to usr/bin
```
Qué observar: `Access (0644/-rw-r--r--)` muestra los permisos en octal y simbólico a la vez (Día 3). En UTM, `/usr/bin/ls` dice `ARM aarch64`.

8. Enlaces duros y simbólicos.

```bash
echo "Politica de respaldos v1" > politica.txt
ln politica.txt politica-hard.txt
ln -s /home/student/empresa/documentos/politica.txt politica-link.txt
ln -s /home/student/empresa/logs ~/logs-empresa
ls -li politica*
ls -l ~/logs-empresa
```
Salida esperada:
```
17512440 -rw-r--r--. 2 student student 25 ... politica-hard.txt
17512441 lrwxrwxrwx. 1 student student 45 ... politica-link.txt -> /home/student/empresa/documentos/politica.txt
17512440 -rw-r--r--. 2 student student 25 ... politica.txt
lrwxrwxrwx. 1 student student 26 ... /home/student/logs-empresa -> /home/student/empresa/logs
```
Qué observar: `politica.txt` y `politica-hard.txt` comparten inodo y tienen `2` en la columna de enlaces; el simbólico tiene inodo propio, tipo `l`, tamaño = longitud de la ruta (45 caracteres) y muestra `->`. `ls` ordena con las reglas del idioma (`LANG`), que ignoran el guion y el punto: por eso `politica-link.txt` sale antes que `politica.txt`.

9. Qué pasa al borrar el original.

```bash
echo "Politica de respaldos v2" >> politica-hard.txt
cat politica.txt
rm politica.txt
cat politica-hard.txt
cat politica-link.txt
ls -l politica-link.txt
mv politica-hard.txt politica.txt
cat politica-link.txt
```
Salida esperada:
```
Politica de respaldos v1
Politica de respaldos v2
Politica de respaldos v1
Politica de respaldos v2
cat: politica-link.txt: No such file or directory
lrwxrwxrwx. 1 student student 45 ... politica-link.txt -> /home/student/empresa/documentos/politica.txt
Politica de respaldos v1
Politica de respaldos v2
```
Qué observar: escribir por un nombre se ve por el otro (mismo inodo). Al borrar `politica.txt` el duro conserva los datos y el simbólico queda roto (en rojo con `ls --color`); al recrear el nombre, el simbólico "revive".

10. Globbing.

```bash
touch archivo{a..c}.txt
ls informe*
ls informe?.txt
ls informe[1-2].txt
ls *.{txt,csv}
ls [!i]*
echo ~/empresa/*/
```
Salida esperada:
```
informe1.txt  informe2.txt  informe3.txt  informe4.txt
informe1.txt  informe2.txt  informe3.txt  informe4.txt
informe1.txt  informe2.txt
archivoa.txt  archivob.txt  archivoc.txt  borrador.txt  informe1.txt  informe2.txt  informe3.txt  informe4.txt  politica-link.txt  politica.txt  presupuesto.csv
archivoa.txt  archivob.txt  archivoc.txt  borrador.txt  plan-2026.doc  politica-link.txt  politica.txt  presupuesto.csv
/home/student/empresa/backups/ /home/student/empresa/clientes/ /home/student/empresa/documentos/ /home/student/empresa/logs/
```
Qué observar: `echo` con un patrón muestra exactamente lo que recibiría cualquier otro comando; conviene usarlo antes de un `rm`.

11. Tipos de archivo en `ls -l`.

```bash
ls -l /dev/null /dev/tty /etc/hostname /bin /etc
ls -l /dev/sda /dev/vda 2>/dev/null
```
Salida esperada:
```
lrwxrwxrwx. 1 root root       7 ... /bin -> usr/bin
crw-rw-rw-. 1 root root    1, 3 ... /dev/null
crw-rw-rw-. 1 root tty     5, 0 ... /dev/tty
-rw-r--r--. 1 root root       7 ... /etc/hostname

/etc:
total 1284
...
brw-rw----. 1 root disk 8, 0 ... /dev/sda
```
Qué observar: `l`, `c`, `-`, `d` (por eso `/etc` lista su contenido: `-d` lo evitaría) y `b`. Los dispositivos muestran `mayor, menor` en vez de tamaño. En UTM sale `/dev/vda` con `252, 0` (o `253, 0`: el número mayor de virtio se asigna dinámicamente).

12. Borrar con cuidado.

```bash
rm -i archivoa.txt
rm archivo?.txt
mkdir vacio && rmdir vacio
rmdir ~/empresa/backups/documentos
ls ~/empresa/backups/documentos
rm -r ~/empresa/backups/documentos
ls ~/empresa/backups
```
Salida esperada:
```
rm: remove regular file 'archivoa.txt'? y
rmdir: failed to remove '/home/student/empresa/backups/documentos': Directory not empty
copia-informe1.txt  informe1.txt  informe2.txt  informe3.txt  informe4.txt  informe5.txt  plan-2026.doc  presupuesto.csv
```
Qué observar: la copia conserva `copia-informe1.txt` e `informe5.txt` porque se hizo (paso 4) antes de renombrar y mover (paso 6): una copia es una foto del momento. El `ls` antes del `rm -r` es el hábito que se quiere formar. El último `ls` no muestra nada: `backups` quedó vacío (lo llenaremos en el Bloque 5).

13. Demostración del instructor (no la ejecuta nadie): en una terminal escribir `rm -rf ~/empresa /backups` y NO pulsar Enter; preguntar al grupo qué borraría. Respuesta: `~/empresa` completo y luego intentaría `/backups`. Un espacio de diferencia.

- **Checkpoint:** pegar en el chat la salida de:

```bash
tree ~/empresa && ls -li ~/empresa/documentos/politica*
```

Salida esperada:
```
/home/student/empresa
├── backups
├── clientes
│   ├── cliente-acme.txt
│   └── informe2.txt
├── documentos
│   ├── borrador.txt
│   ├── informe1.txt
│   ├── informe2.txt
│   ├── informe3.txt
│   ├── informe4.txt
│   ├── plan-2026.doc
│   ├── politica-link.txt -> /home/student/empresa/documentos/politica.txt
│   ├── politica.txt
│   └── presupuesto.csv
└── logs

4 directories, 11 files
17512441 lrwxrwxrwx. 1 student student 45 ... politica-link.txt -> /home/student/empresa/documentos/politica.txt
17512440 -rw-r--r--. 1 student student 50 ... politica.txt
```

---

## Bloque 3 — Texto, redirección, pipes y grep

### Conceptos (15 min)

**Ver texto.**
- `cat archivo` vuelca todo (`-n` numera). Solo para archivos cortos.
- `less archivo` pagina: `Espacio`/`b` avanza/retrocede una página, `g`/`G` inicio/fin, `/texto` busca hacia adelante (`n` siguiente, `N` anterior), `&texto` muestra solo las líneas que coinciden, `-N` numera líneas, `F` sigue el archivo como `tail -f` (`Ctrl+C` para volver), `q` sale. `man` usa `less`, así que todo esto vale también para los manuales.
- `head -n 5` / `head -5` primeras líneas; `tail -n 5` últimas; `tail -f` se queda esperando y muestra lo que se vaya añadiendo: es la forma de "ver un log en vivo".
- `wc` cuenta: `-l` líneas, `-w` palabras, `-c` bytes.
- `nl` numera líneas; `tr` traduce o borra caracteres (`tr 'a-z' 'A-Z'`, `tr -d '\r'` para quitar retornos de carro de archivos hechos en Windows); `diff a b` muestra las diferencias (`-u` formato unificado, `-r` compara directorios).

**Ordenar y resumir.**
- `sort` ordena alfabéticamente; `-n` numérico, `-r` inverso, `-u` sin duplicados, `-k N` por el campo N, `-t :` cambia el separador de campos, `-h` tamaños humanos (`du -sh * | sort -h`).
- `uniq` elimina líneas repetidas *consecutivas*; `-c` las cuenta. Por eso casi siempre va precedido de `sort`. El patrón `sort | uniq -c | sort -rn` = "top de lo más repetido" es el más útil del día.
- `cut -d' ' -f3` extrae el campo 3 usando espacio como separador; `-f1,4` varios campos; `-f6-` del 6 al final; `-c1-10` por columnas de caracteres.

**Los tres flujos.** Cada programa tiene una entrada estándar (stdin, 0), una salida estándar (stdout, 1) y una salida de error (stderr, 2). La shell permite redirigirlos:

| Sintaxis | Efecto |
|---|---|
| `cmd > archivo` | stdout a archivo (lo **sobrescribe**) |
| `cmd >> archivo` | stdout al final del archivo |
| `cmd 2> archivo` | stderr a archivo |
| `cmd > salida.txt 2> errores.txt` | separados |
| `cmd > todo.txt 2>&1` | ambos al mismo archivo (el orden importa: primero `>`, luego `2>&1`) |
| `cmd &> todo.txt` | igual, forma corta de bash |
| `cmd 2> /dev/null` | descartar los errores (el "agujero negro") |
| `cmd < archivo` | stdin desde archivo |
| `cmd1 \| cmd2` | pipe: stdout de cmd1 pasa a stdin de cmd2 |
| `cmd \| tee archivo` | guarda una copia y sigue mostrando por pantalla (`-a` añade) |

Dos advertencias que hay que decir en voz alta:
1. `>` borra el contenido anterior sin preguntar. `set -o noclobber` obliga a usar `>|` para sobrescribir.
2. `sudo echo "x" > /etc/archivo` **falla** con Permission denied: `sudo` aplica a `echo`, pero la redirección la hace la shell del usuario. La forma correcta es `echo "x" | sudo tee /etc/archivo` (o `sudo tee -a` para añadir).

**`xargs`** convierte líneas de stdin en argumentos de otro comando: `find ... | xargs wc -l`. `-n 1` un argumento por ejecución; `-I {}` coloca el argumento donde se indique. Para nombres con espacios: `find -print0 | xargs -0`.

**`grep`** busca líneas que coincidan con un patrón:
- `-i` ignora mayúsculas, `-v` invierte (las que NO coinciden), `-n` número de línea, `-c` cuenta, `-w` palabra completa, `-r` recursivo en directorios, `-l` solo nombres de archivo, `-o` solo la parte que coincide, `-A 2`/`-B 2` líneas de contexto después/antes, `-E` expresiones regulares extendidas.
- Regex básicas: `^` inicio de línea, `$` fin de línea, `.` cualquier carácter, `*` cero o más del anterior, `[0-9]` un dígito, `[^a]` cualquiera menos a. Con `-E`: `|` alternativa (`grep -E "ERROR|WARN"`), `+` uno o más, `{4}` exactamente cuatro, `()` agrupar.
- Ejemplos que hay que mostrar: `grep -E "^2026-03-1[0-9]"` (días 10 a 19), `grep -E "denegado$"`, `grep -v "^#"` (quitar comentarios), `grep -v "^$"` (quitar líneas vacías), `grep -Ev "^#|^$" /etc/ssh/sshd_config` (configuración efectiva sin ruido). Este último es el que más se usa en soporte.

### Lab 3.1 — Analizar un log de 300 líneas (30 min)

- **Objetivo:** generar un log realista y responder preguntas de soporte con `grep`, `cut`, `sort`, `uniq`, `wc` y redirección.
- **Ritmo:** núcleo = pasos 1, 3, 4, 5, 7, 8 y 11. Los pasos 2 y 6 se pueden acortar; 9 y 10 (`xargs`, `diff`, `tail -f` con subshell) son demo del instructor si falta tiempo.

1. Crear el generador y ejecutarlo. Se pega el bloque completo tal cual (el `cat <<'FIN'` crea el archivo; se explica en el Día 7, hoy solo se usa). La semilla `RANDOM=124` hace que todos los participantes obtengan el mismo archivo.

```bash
cat > ~/empresa/genera-log.sh <<'FIN'
#!/bin/bash
# genera-log.sh: crea un log de prueba de N lineas (300 por defecto)
# Formato: FECHA HORA NIVEL USUARIO SERVICIO MENSAJE
RANDOM=124
LINEAS=${1:-300}
niveles=(INFO INFO INFO WARN ERROR FAIL)
usuarios=(ana carlos pedro root backup)
servicios=(sshd httpd crond nfs-server backup.sh)
mensajes=("Sesion iniciada" "Sesion cerrada" "Conexion rechazada" "Archivo no encontrado" "Tarea completada" "Tiempo de espera agotado" "Permiso denegado" "Disco al 90 por ciento")
for i in $(seq 1 "$LINEAS"); do
  printf "2026-03-%02d %02d:%02d:%02d %s %s %s %s\n" \
    $((RANDOM % 28 + 1)) $((RANDOM % 24)) $((RANDOM % 60)) $((RANDOM % 60)) \
    "${niveles[RANDOM % 6]}" "${usuarios[RANDOM % 5]}" \
    "${servicios[RANDOM % 5]}" "${mensajes[RANDOM % 8]}"
done
FIN
bash ~/empresa/genera-log.sh > ~/empresa/logs/servidor.log
wc -l ~/empresa/logs/servidor.log
md5sum ~/empresa/logs/servidor.log
head -3 ~/empresa/logs/servidor.log
```
Salida esperada:
```
300 /home/student/empresa/logs/servidor.log
3f9c...  /home/student/empresa/logs/servidor.log
2026-03-17 04:51:09 INFO carlos httpd Tarea completada
2026-03-02 13:08:44 WARN ana sshd Conexion rechazada
2026-03-24 21:33:15 INFO root crond Sesion cerrada
```
Qué observar: si el `md5sum` coincide con el del instructor, todos tienen exactamente el mismo archivo y los números que siguen coincidirán. La secuencia de `$RANDOM` depende de la versión de bash (bash 5.1 cambió el generador), pero como todas las VM del curso son RHEL 9 con la misma versión de bash (`bash --version`), el archivo debe salir idéntico. ⚠️ Verificar en la VM antes de la clase: generar el log, anotar el `md5sum` real y recalcular las cifras de los pasos 3 a 6; **los valores de las salidas siguientes son ilustrativos**. Si a alguien no le coincide, no importa: los comandos son los mismos y solo cambian las cifras.

2. Ver el archivo de varias formas.

```bash
cd ~/empresa/logs
cat servidor.log | head -5
less servidor.log
```
En `less`: `G` al final, `g` al inicio, `/FAIL` y `n` varias veces, `&ERROR` para filtrar, `q`.

```bash
head -n 3 servidor.log
tail -n 3 servidor.log
nl servidor.log | tail -2
wc -l -w servidor.log
```
Salida esperada (resumida):
```
   299	2026-03-11 07:14:58 ERROR pedro nfs-server Permiso denegado
   300	2026-03-05 18:40:22 INFO backup backup.sh Tarea completada
 300 2325 servidor.log
```

3. `grep` básico: contar, invertir, numerar, palabra completa.

```bash
grep -c ERROR servidor.log
grep -c -w FAIL servidor.log
grep -v INFO servidor.log | wc -l
grep -n FAIL servidor.log | head -3
grep -i "disco" servidor.log | wc -l
grep -w ana servidor.log | head -2
```
Salida esperada (ilustrativa):
```
51
48
152
7:2026-03-09 02:17:41 FAIL pedro httpd Tiempo de espera agotado
12:2026-03-21 11:05:03 FAIL ana crond Archivo no encontrado
19:2026-03-03 23:48:30 FAIL root sshd Conexion rechazada
36
2026-03-02 13:08:44 WARN ana sshd Conexion rechazada
2026-03-21 11:05:03 FAIL ana crond Archivo no encontrado
```
Qué observar: `-w ana` evita coincidir con palabras que contengan "ana". `-v INFO` da todo lo que no es informativo.

4. `grep -E` con expresiones regulares.

```bash
grep -E "ERROR|FAIL" servidor.log | wc -l
grep -E "^2026-03-1[0-9]" servidor.log | wc -l
grep -E "denegado$" servidor.log | wc -l
grep -E "(ERROR|FAIL) .* sshd" servidor.log | head -3
grep -o -E "INFO|WARN|ERROR|FAIL" servidor.log | sort | uniq -c
```
Salida esperada (ilustrativa):
```
99
104
41
2026-03-03 23:48:30 FAIL root sshd Conexion rechazada
2026-03-14 06:20:11 ERROR carlos sshd Permiso denegado
2026-03-22 15:02:57 FAIL ana sshd Tiempo de espera agotado
     51 ERROR
     48 FAIL
    148 INFO
     53 WARN
```
Qué observar: `^` ancla al inicio (fecha), `$` al final (mensaje); `-o` extrae solo lo que coincide, lo que permite contar sin `cut`.

5. `cut` + `sort` + `uniq`: los "top" de soporte.

```bash
cut -d' ' -f3 servidor.log | sort | uniq -c | sort -rn
cut -d' ' -f4 servidor.log | sort | uniq -c | sort -rn
cut -d' ' -f5 servidor.log | sort -u
grep -E "ERROR|FAIL" servidor.log | cut -d' ' -f5 | sort | uniq -c | sort -rn | head -3
cut -d' ' -f2 servidor.log | cut -d: -f1 | sort | uniq -c | sort -rn | head -3
cut -d' ' -f6- servidor.log | sort | uniq -c | sort -rn
```
Salida esperada (ilustrativa):
```
    148 INFO
     53 WARN
     51 ERROR
     48 FAIL
     66 root
     62 ana
     60 pedro
     58 carlos
     54 backup
backup.sh
crond
httpd
nfs-server
sshd
     24 httpd
     21 sshd
     19 crond
     17 03
     16 14
     15 21
     43 Tarea completada
     41 Permiso denegado
     ...
```
Qué observar: `uniq -c` sin `sort` previo daría cuentas parciales. El segundo `sort -rn` ordena por la cuenta. `-f6-` toma el mensaje completo aunque tenga espacios.

6. Ordenar por fecha, por campo y numéricamente.

```bash
sort -k1,2 servidor.log | head -2
sort -k1,2 servidor.log | tail -1
sort -t: -k3 -n /etc/passwd | tail -3
cut -d: -f3 /etc/passwd | sort -n | tail -3
```
Salida esperada (ilustrativa):
```
2026-03-01 00:47:13 INFO pedro httpd Sesion iniciada
2026-03-01 03:12:55 WARN backup backup.sh Disco al 90 por ciento
2026-03-28 23:51:02 INFO ana nfs-server Sesion cerrada
sssd:x:998:996:User for sssd:/:/sbin/nologin
student:x:1000:1000:student:/home/student:/bin/bash
nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
998
1000
65534
```
Qué observar: `-k1,2` ordena por fecha y hora (como el formato es AAAA-MM-DD el orden alfabético coincide con el cronológico). En `/etc/passwd` el separador es `:` y el campo 3 es el UID, que hay que ordenar numéricamente (`-n`) para que 1000 no quede antes que 65534.

7. Redirección: guardar resultados, separar errores, descartar.

```bash
grep ERROR servidor.log > errores.txt
grep FAIL servidor.log >> errores.txt
wc -l errores.txt
ls /etc/hostname /noexiste > salida.txt 2> errores-cmd.txt
cat salida.txt; cat errores-cmd.txt
ls /etc/hostname /noexiste > todo.txt 2>&1; cat todo.txt
ls /etc/hostname /noexiste 2>&1 > solo-stdout.txt
ls /noexiste 2> /dev/null; echo "codigo=$?"
wc -l < servidor.log
tr 'a-z' 'A-Z' < servidor.log | head -1
```
Salida esperada (ilustrativa):
```
99 errores.txt
/etc/hostname
ls: cannot access '/noexiste': No such file or directory
ls: cannot access '/noexiste': No such file or directory
/etc/hostname
ls: cannot access '/noexiste': No such file or directory
codigo=2
300
2026-03-17 04:51:09 INFO CARLOS HTTPD TAREA COMPLETADA
```
Qué observar: con `2>&1 > archivo` (orden invertido) el error salió por pantalla y solo stdout fue al archivo. `wc -l < archivo` no muestra el nombre porque `wc` leyó de stdin. `/dev/null` silenció el error pero el código de salida sigue siendo 2.

8. `tee`, y el problema de `sudo` con redirección.

```bash
grep -E "ERROR|FAIL" servidor.log | tee criticos.txt | wc -l
sudo echo "Servidor rhel01 - PGN Direccion de Informatica" > /etc/motd
echo "Servidor rhel01 - PGN Direccion de Informatica" | sudo tee /etc/motd
cat /etc/motd
```
Salida esperada:
```
99
-bash: /etc/motd: Permission denied
Servidor rhel01 - PGN Direccion de Informatica
Servidor rhel01 - PGN Direccion de Informatica
```
Qué observar: `tee` escribió `criticos.txt` y a la vez pasó las líneas a `wc`. El `sudo echo >` falla; `| sudo tee` funciona. El mensaje de `/etc/motd` aparecerá en el próximo inicio de sesión SSH.

9. `xargs`, `diff`.

```bash
ls ~/empresa/documentos/*.txt | xargs wc -l | tail -1
ls ~/empresa/documentos/*.txt | xargs wc -l | wc -l
ls ~/empresa/documentos | xargs -n 1 echo "Archivo:" | head -3
cp servidor.log copia.log; echo "linea extra" >> copia.log
diff servidor.log copia.log
diff -r ~/empresa/documentos ~/empresa/clientes | head -3
```
Salida esperada:
```
  9 total
8
Archivo: borrador.txt
Archivo: informe1.txt
Archivo: informe2.txt
300a301
> linea extra
Only in /home/student/empresa/documentos: borrador.txt
Only in /home/student/empresa/clientes: cliente-acme.txt
Only in /home/student/empresa/documentos: informe1.txt
```
Qué observar: el patrón `*.txt` expande a 7 nombres (`borrador`, `informe1`–`informe4`, `politica.txt`, `politica-link.txt`; `plan-2026.doc` y `presupuesto.csv` no entran) y `wc` corrió una sola vez con los 7: 7 líneas más la línea `total` = 8. El total de 9 líneas sale de 1+1+1+1+1+2+2 (el enlace simbólico se cuenta como su destino). `300a301` se lee "después de la línea 300 del primero, añadir la 301 del segundo". `diff -r` recorre los nombres en orden alfabético mezclando ambos directorios. Sin diferencias, `diff` no imprime nada y devuelve 0: así se verifica una restauración.

10. `tail -f` en una sola terminal (truco: una subshell en segundo plano escribe 5 segundos después).

```bash
touch en-vivo.log
( sleep 5; echo "$(date '+%F %T') WARN root sshd Intento de acceso" >> en-vivo.log ) &
tail -f en-vivo.log
```
Salida esperada:
```
[1] 4321
(pasan 5 segundos)
2026-09-03 10:05:41 WARN root sshd Intento de acceso
```
Salir con `Ctrl+C`; al volver el prompt aparece `[1]+  Done   ( sleep 5; echo ... >> en-vivo.log )`. Qué observar: `tail -f` no termina solo; se queda "escuchando". Quien tenga una segunda terminal puede repetirlo escribiendo desde la otra.

11. Logs reales del sistema (requieren `sudo`).

```bash
ls -l /var/log/messages /var/log/secure
tail -3 /var/log/messages
sudo tail -3 /var/log/messages
sudo grep -ic error /var/log/messages
sudo grep -i "Accepted" /var/log/secure | tail -3
sudo grep -Ev "^#|^$" /etc/ssh/sshd_config
```
Salida esperada (resumida):
```
-rw-------. 1 root root 812345 ... /var/log/messages
-rw-------. 1 root root  45678 ... /var/log/secure
tail: cannot open '/var/log/messages' for reading: Permission denied
Sep  3 10:04:02 rhel01 systemd[1]: Started dnf makecache.
...
14
Sep  3 09:02:11 rhel01 sshd[1523]: Accepted password for student from 10.0.2.2 port 55012 ssh2
...
Include /etc/ssh/sshd_config.d/*.conf
AuthorizedKeysFile	.ssh/authorized_keys
Subsystem	sftp	/usr/libexec/openssh/sftp-server
```
Qué observar: los logs del sistema son `600 root`; siempre `sudo`. `10.0.2.2` es la puerta NAT del hipervisor: así se ve la conexión SSH desde el equipo propio. El `grep -Ev "^#|^$"` sobre un archivo de configuración deja solo lo activo.

- **Checkpoint:** pegar en el chat la salida de:

```bash
wc -l ~/empresa/logs/servidor.log ~/empresa/logs/errores.txt; cut -d' ' -f3 ~/empresa/logs/servidor.log | sort | uniq -c | sort -rn
```

---

## Bloque 4 — Búsquedas avanzadas

### Conceptos (10 min)

**`find`** recorre el árbol en tiempo real y aplica criterios. Sintaxis: `find DÓNDE CRITERIOS ACCIÓN`.

| Criterio | Significado |
|---|---|
| `-name "*.conf"` / `-iname` | Por nombre (con comillas para que la shell no expanda) / sin distinguir mayúsculas |
| `-type f` / `d` / `l` | Archivo regular / directorio / enlace simbólico |
| `-size +100k` / `-size -1M` / `-size +1G` | Mayor que / menor que (k, M, G) |
| `-mtime -1` / `-mtime +7` | Modificado hace menos de 1 día / hace más de 7 días |
| `-mmin -60` | Modificado en los últimos 60 minutos |
| `-user student` / `-group wheel` | Por dueño / grupo |
| `-perm 000` / `-perm -4000` | Permisos exactos / que incluyan esos bits (setuid) |
| `-empty` | Archivos vacíos o directorios vacíos |
| `-newer archivo` | Modificado después que `archivo` |
| `-maxdepth 1` | No bajar de nivel (va justo después de la ruta) |
| `\( -name "*.txt" -o -name "*.csv" \)` | Combinación con OR |

| Acción | Significado |
|---|---|
| (ninguna) | Imprime la ruta (`-print`) |
| `-ls` | Muestra estilo `ls -l` |
| `-exec cmd {} \;` | Ejecuta `cmd` una vez por cada resultado (`{}` = la ruta) |
| `-exec cmd {} +` | Ejecuta `cmd` una vez con todos los resultados como argumentos (mucho más rápido) |
| `-ok cmd {} \;` | Como `-exec` pero pregunta antes de cada uno |
| `-delete` | Borra. Debe ir al **final**; se prueba antes sin `-delete` para ver qué borraría |

Hay que decirlo: `find / -name x` tarda y lanza `Permission denied` en directorios ajenos; se limita la ruta o se añade `2>/dev/null`; con `sudo` se ve todo.

**`locate`** busca en una base de datos precalculada: instantáneo, pero solo conoce lo que había en el último `updatedb` (se ejecuta a diario con un timer). En RHEL 9 el paquete es `mlocate`; si `dnf` dijera que no existe, se prueba `plocate` (mismo uso). Tras instalar hay que correr `sudo updatedb` una vez. `locate -i` ignora mayúsculas.

**`which`, `whereis`, `type`.** `which cmd` dice qué ejecutable del PATH se usaría; `whereis` añade la página de manual y fuentes; `type` es el más completo porque también identifica alias y builtins. Si `which` no encuentra algo que "debería estar", casi siempre es que el directorio no está en el PATH (`/usr/sbin` para usuarios no root en sistemas antiguos) o que el paquete no está instalado.

### Lab 4.1 — `find` en `/etc` y `/var/log`; `locate` (15 min)

- **Objetivo:** localizar archivos por nombre, tipo, tamaño, fecha, dueño y permisos, y actuar sobre ellos con `-exec`.
- **Ritmo:** núcleo = pasos 1, 2, 3, 5 y 6. Los pasos 7 (`locate`) y 8 (`which`/`whereis`/`type`) están en "Si sobra tiempo": se hacen como demo del instructor o quedan de tarea (`mlocate` se instala en 30 segundos en casa).

1. Por nombre y tipo en `/etc`.

```bash
sudo find /etc -name "*.conf" | wc -l
sudo find /etc -iname "*.CONF" | wc -l
sudo find /etc -maxdepth 1 -type d -name "*.d" | head -5
sudo find /etc -type l | head -3
find /etc -name "*.conf" 2>&1 | grep -c "Permission denied"
```
Salida esperada (ilustrativa; el orden de `find` es el del disco, no alfabético):
```
262
262
/etc/rc.d
/etc/sysctl.d
/etc/modprobe.d
/etc/pam.d
/etc/profile.d
/etc/mtab
/etc/localtime
/etc/system-release
9
```
Qué observar: sin `sudo`, algunos directorios de `/etc` no se pueden leer; por eso la última línea cuenta errores. `-iname` da lo mismo que `-name` aquí porque no hay `.CONF` en mayúsculas. Los enlaces de `/etc` típicos son `mtab -> ../proc/self/mounts`, `localtime -> ../usr/share/zoneinfo/America/Panama`, `system-release -> redhat-release` y `os-release -> ../usr/lib/os-release`: salen con `-type l` y no con `-type d` aunque apunten a un directorio, porque `find` no sigue enlaces salvo con `-L`.

2. Por tamaño en `/etc` y `/var/log`.

```bash
sudo find /etc -type f -size +100k -exec ls -lh {} \;
sudo find /var/log -type f -size +1M
```
Salida esperada (ilustrativa):
```
-rw-r--r--. 1 root root 692K ... /etc/services
-rw-r--r--. 1 root root  11M ... /etc/udev/hwdb.bin
-rw-r--r--. 1 root root 200K ... /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
...
/var/log/anaconda/journal.log
/var/log/lastlog
/var/log/messages
```
Qué observar: `-exec ls -lh {} \;` ejecuta un `ls` por archivo. `/var/log/lastlog` parece enorme (unos 19 MB) pero es un archivo *sparse*: `du -h /var/log/lastlog` mostrará pocos KB reales. Sobre `/var/log/journal/`: si el directorio no existe, el journal de systemd es volátil (vive en `/run/log/journal`) y se hará persistente el Día 4; si existe, ya es persistente y el Día 4 solo se comprueba. ⚠️ Verificar en la VM antes de la clase con `ls -d /var/log/journal` y `journalctl --disk-usage`.

3. Por fecha: qué cambió recientemente.

```bash
sudo find /etc -type f -mmin -120 | head
sudo find /var/log -type f -mtime -1 | head -5
find ~/empresa -newer ~/empresa/documentos/informe1.txt -type f
```
Salida esperada (ilustrativa):
```
/etc/motd
/etc/ld.so.cache
/etc/.updated
/var/log/dnf.log
/var/log/dnf.rpm.log
/var/log/hawkey.log
/var/log/messages
/var/log/secure
/home/student/empresa/documentos/politica.txt
/home/student/empresa/logs/servidor.log
...
```
Qué observar: `/etc/motd` aparece porque lo modificamos con `tee` hace un rato. "¿Qué cambió en `/etc` en las últimas 2 horas?" es una de las preguntas más útiles de un troubleshooting.

4. Por dueño, permisos y vacíos.

```bash
find /home -user student -type f | wc -l
find ~/empresa -group student -type d
sudo find /etc -type f -perm 000
find /usr/bin -perm -4000 -type f | head -5
find ~/empresa -empty
```
Salida esperada (ilustrativa):
```
24
/home/student/empresa
/home/student/empresa/backups
/home/student/empresa/clientes
/home/student/empresa/documentos
/home/student/empresa/logs
/etc/gshadow
/etc/shadow
/etc/gshadow-
/etc/shadow-
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/home/student/empresa/backups
/home/student/empresa/documentos/plan-2026.doc
/home/student/empresa/documentos/presupuesto.csv
```
Qué observar: en RHEL `/etc/shadow` tiene permisos `000`: ni siquiera root lo lee por permisos, sino porque root ignora los permisos. Los ejecutables con setuid (`-perm -4000`) se retoman el Día 3.

5. `-exec` con `\;` vs `+`, y combinaciones.

```bash
find ~/empresa/documentos -name "*.txt" -exec wc -l {} \;
find ~/empresa/documentos -name "*.txt" -exec wc -l {} +
find ~/empresa/documentos \( -name "*.txt" -o -name "*.csv" \) -type f | wc -l
find ~/empresa -type f -exec grep -l FAIL {} +
sudo find /var/log -name "*.log" -exec wc -l {} + | sort -n | tail -3
```
Salida esperada (ilustrativa; el orden de `find` puede variar):
```
1 /home/student/empresa/documentos/informe2.txt
...
   1 /home/student/empresa/documentos/borrador.txt
   1 /home/student/empresa/documentos/informe1.txt
   ...
   9 total
7
/home/student/empresa/genera-log.sh
/home/student/empresa/logs/servidor.log
/home/student/empresa/logs/errores.txt
/home/student/empresa/logs/criticos.txt
/home/student/empresa/logs/copia.log
   210 /var/log/dnf.rpm.log
  1380 /var/log/hawkey.log
  4200 total
```
Qué observar: con `\;` cada `wc` corre aparte (sin línea `total`); con `+` corre una sola vez y da el total (9 líneas, igual que con `xargs` en el Lab 3.1). El `7` son 6 `.txt` regulares más `presupuesto.csv`: `-type f` excluye el enlace `politica-link.txt`. El paréntesis escapado agrupa el OR. `genera-log.sh` aparece en el `grep -l FAIL` porque contiene la palabra en su lista de niveles: `grep` no distingue un log de un script.

6. `-delete` con precaución: primero ver, luego borrar.

```bash
mkdir -p ~/empresa/tmp && touch ~/empresa/tmp/{a,b,c}.tmp ~/empresa/tmp/importante.txt
find ~/empresa/tmp -name "*.tmp"
find ~/empresa/tmp -name "*.tmp" -delete
ls ~/empresa/tmp
find ~/empresa/tmp -type f -ok rm {} \;
```
Salida esperada:
```
/home/student/empresa/tmp/a.tmp
/home/student/empresa/tmp/b.tmp
/home/student/empresa/tmp/c.tmp
importante.txt
< rm ... /home/student/empresa/tmp/importante.txt > ? n
```
Qué observar: responder `n` en el `-ok`. Advertir: `find dir -delete -name "*.tmp"` (con `-delete` antes del criterio) borra TODO `dir`. `-delete` siempre al final y siempre después de un `find` de prueba.

7. `locate`.

```bash
sudo dnf install -y mlocate
locate hostname
sudo updatedb
locate hostname | head -4
locate -i SERVIDOR.LOG
touch ~/empresa/nuevo.txt; locate nuevo.txt; sudo updatedb; locate nuevo.txt
systemctl list-timers | grep -i locate
```
Salida esperada:
```
...
Complete!
locate: can not stat () `/var/lib/mlocate/mlocate.db': No such file or directory
/etc/hostname
/usr/bin/hostname
/usr/lib/systemd/system/systemd-hostnamed.service
/usr/lib64/gconv/... (o similar)
/home/student/empresa/logs/servidor.log

/home/student/empresa/nuevo.txt
Thu 2026-09-04 00:00:00 -05  13h left  ...  mlocate-updatedb.timer  mlocate-updatedb.service
```
Qué observar: recién instalado no hay base de datos; tras `updatedb` es instantáneo; un archivo nuevo no aparece hasta el siguiente `updatedb` (el timer lo hace a diario). ⚠️ Verificar en la VM antes de la clase: si `systemctl list-timers` no muestra `mlocate-updatedb.timer`, la actualización diaria va por `cron` (`ls -l /etc/cron.daily/mlocate`); el resto del paso no cambia. Si `dnf install mlocate` respondiera `No match for argument`, usar `sudo dnf install -y plocate` (los comandos `locate`/`updatedb` son los mismos; la base queda en `/var/lib/plocate/`).

8. `which`, `whereis`, `type`.

```bash
which tar; whereis tar; type tar
type ll; type cd; type -t ls
which noexiste; echo $?
```
Salida esperada:
```
/usr/bin/tar
tar: /usr/bin/tar /usr/share/man/man1/tar.1.gz
tar is /usr/bin/tar
ll is aliased to `ls -l --color=auto'
cd is a shell builtin
alias
/usr/bin/which: no noexiste in (/home/student/.local/bin:/home/student/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin)
1
```

- **Checkpoint:** pegar en el chat la salida de:

```bash
sudo find /etc -type f -perm 000 | sort; find ~/empresa -name "*.txt" -exec wc -l {} + | tail -1; find ~/empresa/tmp -type f
```
(Quien haya hecho el paso 7 añade también `locate servidor.log`.)

---

## Bloque 5 — Compresión y empaquetado

### Conceptos (5 min)

- **Empaquetar** (juntar muchos archivos en uno, conservando rutas, permisos y dueños) lo hace `tar`. **Comprimir** (reducir tamaño) lo hacen `gzip`, `bzip2` y `xz`. Por eso un respaldo es `.tar.gz`: primero tar, luego gzip. `zip` hace ambas cosas y es el formato que entienden los usuarios de Windows.
- Opciones de `tar` que hay que memorizar: `c` crear, `x` extraer, `t` listar, `v` detallado, `f archivo` (siempre el último de las letras, porque le sigue el nombre), `z` gzip, `j` bzip2, `J` xz, `-C dir` cambiar a `dir` antes de actuar (para extraer en otro sitio o para no guardar rutas largas), `--exclude=patrón`. Mnemotecnia: **c**rear, e**x**traer, lis**t**ar.
- `tar` quita la `/` inicial de las rutas (`Removing leading '/'`): es una protección para que al restaurar no se sobrescriba el sistema por accidente. Al extraer en `/tmp/restaurar` se obtiene `/tmp/restaurar/etc/...`.
- Al extraer, `tar` moderno detecta la compresión solo: `tar -xf archivo.tar.gz` funciona; `-z` sigue siendo obligatorio al crear (o usar `-a`, que elige por la extensión).
- Compresores: `gzip` rápido y universal; `bzip2` comprime más y tarda más; `xz` comprime aún más y es el más lento (es el habitual en los `.tar.xz` de código fuente; los RPM de RHEL 9 usan `zstd`). Todos reemplazan el archivo original salvo con `-k`. Se leen sin descomprimir con `zcat`, `bzcat`, `xzcat` (y `zless`, `zgrep`).
- Para medir: `ls -lh` (tamaño de archivos), `du -sh` (espacio ocupado por un directorio), `gzip -l` (ratio).

### Lab 5.1 — Respaldar `~/empresa` y `/etc` (15 min)

- **Objetivo:** crear, listar, restaurar y verificar respaldos con `tar`, y comparar compresores.
- **Ritmo:** núcleo = pasos 2, 3 y 4 (crear, listar, restaurar, respaldar `/etc`). El paso 1 (comparar compresores) puede reducirse a `gzip -k` + `ls -lh`; los pasos 5 y 6 (`zip`, `du`) son demo si falta tiempo.

1. Un archivo grande de prueba y los tres compresores.

```bash
cd ~/empresa/backups
seq 1 300000 > numeros.txt
ls -lh numeros.txt
gzip -k numeros.txt
bzip2 -k numeros.txt
time xz -k numeros.txt
ls -lh numeros.txt*
gzip -l numeros.txt.gz
zcat numeros.txt.gz | tail -1
file numeros.txt.*
```
Salida esperada (los tamaños exactos varían; importa el orden):
```
-rw-r--r--. 1 student student 1.9M ... numeros.txt

real	0m2.412s
user	0m2.372s
sys	0m0.031s
...
-rw-r--r--. 1 student student 1.9M ... numeros.txt
-rw-r--r--. 1 student student 373K ... numeros.txt.bz2
-rw-r--r--. 1 student student 622K ... numeros.txt.gz
-rw-r--r--. 1 student student  75K ... numeros.txt.xz
         compressed        uncompressed  ratio uncompressed_name
             636027             1988895  68.0% numeros.txt
300000
numeros.txt.bz2: bzip2 compressed data, block size = 900k
numeros.txt.gz:  gzip compressed data, was "numeros.txt", ...
numeros.txt.xz:  XZ compressed data, checksum CRC64
```
Qué observar: sin `-k` el original habría desaparecido. Con este contenido tan repetitivo (números consecutivos) el orden es claro: `xz` (75K) comprime mucho más que `bzip2` (373K) y que `gzip` (622K), y a cambio es el que más tarda; con datos reales las diferencias suelen ser mucho menores. Para descomprimir: `gunzip`, `bunzip2`, `unxz` (o `gzip -d`, `bzip2 -d`, `xz -d`).

2. Empaquetar `~/empresa` con y sin compresión, listar, excluir.

```bash
tar -cvf /tmp/empresa.tar -C ~ empresa > /tmp/tar-lista.txt
head -5 /tmp/tar-lista.txt; wc -l /tmp/tar-lista.txt
tar -czf /tmp/empresa-$(date +%F).tar.gz -C ~ empresa
tar -cjf /tmp/empresa.tar.bz2 -C ~ empresa
tar -cJf /tmp/empresa.tar.xz -C ~ empresa
tar -czf /tmp/empresa-sin-logs.tar.gz --exclude='*.log' --exclude='numeros.txt*' -C ~ empresa
ls -lh /tmp/empresa*
tar -tzf /tmp/empresa-$(date +%F).tar.gz | head -5
tar -tvzf /tmp/empresa-$(date +%F).tar.gz | grep politica
```
Salida esperada (ilustrativa; el orden dentro del `.tar` es el del disco, no alfabético):
```
empresa/
empresa/backups/
empresa/backups/numeros.txt
empresa/backups/numeros.txt.gz
empresa/backups/numeros.txt.bz2
27 /tmp/tar-lista.txt
-rw-r--r--. 1 student student 1.7M ... /tmp/empresa-2026-09-03.tar.gz
-rw-r--r--. 1 student student  20K ... /tmp/empresa-sin-logs.tar.gz
-rw-r--r--. 1 student student 3.1M ... /tmp/empresa.tar
-rw-r--r--. 1 student student 1.5M ... /tmp/empresa.tar.bz2
-rw-r--r--. 1 student student 1.2M ... /tmp/empresa.tar.xz
empresa/
empresa/backups/
empresa/backups/numeros.txt
empresa/backups/numeros.txt.gz
empresa/backups/numeros.txt.bz2
-rw-r--r-- student/student  50 2026-09-03 09:52 empresa/documentos/politica.txt
lrwxrwxrwx student/student   0 2026-09-03 09:52 empresa/documentos/politica-link.txt -> /home/student/empresa/documentos/politica.txt
```
Qué observar: se guarda la lista de `-v` en un archivo en vez de pasarla por `| head -5`, porque `head` cierra la tubería en cuanto tiene 5 líneas y `tar` moriría con SIGPIPE dejando el `.tar` incompleto. `-C ~ empresa` guarda las rutas como `empresa/...` y no `home/student/empresa/...`. El `.tar` sin comprimir pesa lo mismo que el contenido; de los 3.1M, algo más de 1.0M son `numeros.txt.gz/.bz2/.xz`, que ya no reducen más: por eso el `.tar.gz` queda en ~1.7M y no en 620K. Nótese también que `ls` ordena `empresa-2026-...` y `empresa-sin-logs...` antes que `empresa.tar` porque el idioma (`LANG`) ignora el guion y el punto al comparar. Lo que se excluye (`--exclude`) se nota: `empresa-sin-logs` pesa 20K. `-tv` muestra permisos y dueño: `tar` los conserva. El enlace simbólico se guarda como enlace.

3. Restaurar en otro sitio y verificar.

```bash
mkdir -p /tmp/restaurar
tar -xzf /tmp/empresa-$(date +%F).tar.gz -C /tmp/restaurar
ls /tmp/restaurar/empresa
diff -r ~/empresa /tmp/restaurar/empresa && echo "RESTAURACION IDENTICA"
tar -xzf /tmp/empresa-$(date +%F).tar.gz -C /tmp empresa/documentos/politica.txt
cat /tmp/empresa/documentos/politica.txt
```
Salida esperada:
```
backups  clientes  documentos  genera-log.sh  logs  nuevo.txt  tmp
RESTAURACION IDENTICA
Politica de respaldos v1
Politica de respaldos v2
```
Qué observar: `diff -r` sin salida = restauración fiel. `nuevo.txt` solo aparece si se hizo el paso 7 del Lab 4.1 (`locate`), que es opcional; si no está, la lista es la misma sin ese nombre. El último `tar` extrae un solo archivo del paquete (se indica su ruta tal como aparece en `tar -t`).

4. Caso real: respaldar `/etc` antes de tocarlo.

```bash
sudo tar -czf /tmp/etc-$(date +%F).tar.gz /etc
ls -lh /tmp/etc-*.tar.gz
tar -tzf /tmp/etc-$(date +%F).tar.gz | wc -l
tar -tzf /tmp/etc-$(date +%F).tar.gz | grep -E "^etc/(hostname|hosts|fstab)$"
sudo du -sh /etc
```
Salida esperada (ilustrativa):
```
tar: Removing leading `/' from member names
-rw-r--r--. 1 root root 6.2M ... /tmp/etc-2026-09-03.tar.gz
2100
etc/fstab
etc/hostname
etc/hosts
36M	/etc
```
Qué observar: se necesita `sudo` porque hay archivos de `/etc` que solo root lee (`shadow`, `sudoers`). El aviso `Removing leading '/'` no es un error. Puede aparecer también `tar: /etc/xxx: file changed as we read it` (algún archivo cambió mientras se empaquetaba) y entonces `tar` termina con código 1 aunque el respaldo sea válido: se comprueba con `echo $?` y `tar -tzf`. El archivo queda como root pero legible por todos: en producción se restringiría (`chmod 600`), porque contiene `/etc/shadow`.

5. Restaurar `/etc` en `/tmp/restaurar` y comparar.

```bash
sudo tar -xzf /tmp/etc-$(date +%F).tar.gz -C /tmp/restaurar
ls /tmp/restaurar
ls /tmp/restaurar/etc | head -5
diff /etc/hostname /tmp/restaurar/etc/hostname && echo "hostname OK"
sudo diff -rq /etc /tmp/restaurar/etc | head -3
sudo tar -xzf /tmp/etc-$(date +%F).tar.gz -C /tmp/restaurar etc/hosts
```
Salida esperada:
```
empresa  etc
adjtime  aliases  alternatives  anacrontab  audit
hostname OK
```
Qué observar: `diff -rq` mostrará dos o tres líneas de `No such file or directory` por enlaces simbólicos relativos (`/etc/mtab -> ../proc/self/mounts`, `/etc/localtime -> ../usr/share/zoneinfo/...`) que, dentro de `/tmp/restaurar`, apuntan a rutas inexistentes; el resto es idéntico. Se usa `sudo` al extraer para que `tar` conserve los dueños originales (`root`); sin `sudo`, todo quedaría como `student`. En un caso real se copiaría el archivo restaurado a su sitio con `sudo cp -a /tmp/restaurar/etc/hosts /etc/hosts`.

6. `zip`/`unzip` para intercambiar con Windows; espacio ocupado.

```bash
cd ~ && zip -rq /tmp/empresa.zip empresa
unzip -l /tmp/empresa.zip | tail -3
unzip -q /tmp/empresa.zip -d /tmp/restaurar-zip
du -sh ~/empresa /tmp/restaurar /tmp/restaurar-zip
du -sh ~/empresa/* | sort -h
```
Salida esperada (ilustrativa):
```
   1988895  2026-09-03 09:58   empresa/backups/numeros.txt
---------                     -------
   3121458                     27 files
3.1M	/home/student/empresa
39M	/tmp/restaurar
3.1M	/tmp/restaurar-zip
0	/home/student/empresa/nuevo.txt
4.0K	/home/student/empresa/genera-log.sh
4.0K	/home/student/empresa/tmp
8.0K	/home/student/empresa/clientes
...
3.0M	/home/student/empresa/backups
```
Qué observar: `zip` necesita `-r` para directorios; `unzip -d` es el equivalente de `tar -C`. `zip` sin `-y` guarda el enlace simbólico como copia del archivo destino (no como enlace): otra razón para usar `tar` dentro de Linux. `du` mide bloques ocupados (múltiplos de 4K en XFS), por eso un archivo de 31 bytes ocupa 4.0K y solo el vacío da 0. `du -sh dir/* | sort -h` responde "¿qué está ocupando el espacio?" (se usará el Día 6 con `/var`).

- **Checkpoint:** pegar en el chat la salida de:

```bash
ls -lh /tmp/*.tar.gz; ls /tmp/restaurar; diff -r ~/empresa /tmp/restaurar/empresa && echo OK
```

---

## Bloque 6 — Editores: vim y nano

### Conceptos (5 min)

Se usa `vim` porque está en cualquier sistema Linux, porque es lo único garantizado en el examen RHCSA y porque, pasado el primer susto, es más rápido que cualquier editor para tocar configuración. `nano` se enseña como red de seguridad.

**La idea que evita el susto:** vim tiene *modos*. En modo **normal** las teclas son órdenes (moverse, borrar, copiar); en modo **insertar** se escribe texto; en modo **línea de comandos** (`:`) se guarda, se sale y se buscan/reemplazan cosas; en modo **visual** (`v`) se selecciona. Se entra a insertar con `i`; se vuelve a normal con `Esc`. Si hay duda, `Esc` `Esc`.

Lo mínimo para hoy:

| Tecla | Acción |
|---|---|
| `i` / `a` / `o` | Insertar antes del cursor / después / en una línea nueva debajo (`O` arriba, `A` al final de línea) |
| `Esc` | Volver a modo normal |
| `:w` / `:q` / `:wq` (o `:x`) / `:q!` | Guardar / salir / guardar y salir / salir descartando cambios |
| `dd` / `3dd` | Borrar línea / tres líneas (quedan en el portapapeles) |
| `yy` / `p` / `P` | Copiar línea / pegar debajo / pegar arriba |
| `x` / `dw` | Borrar carácter / palabra |
| `u` / `Ctrl+R` | Deshacer / rehacer |
| `/texto` `n` `N` | Buscar / siguiente / anterior |
| `gg` / `G` / `:25` | Ir al inicio / al final / a la línea 25 |
| `0` / `$` / `w` / `b` | Inicio / fin de línea / palabra siguiente / anterior |
| `:set nu` / `:set nonu` | Mostrar / ocultar números de línea |
| `:%s/viejo/nuevo/g` | Reemplazar en todo el archivo (`c` al final pide confirmación por cada uno) |
| `:set paste` | Antes de pegar texto desde fuera por SSH, para que no se desordene la indentación |
| `v` / `V` + movimiento + `y` o `d` | Seleccionar caracteres / líneas y copiar o borrar |

En RHEL 9, `vi` es `vim-minimal` (siempre presente, incluso en modo rescate); `vim-enhanced` añade colores, `vimtutor` y más. Con `vim-enhanced` instalado, al volver a iniciar sesión `vi` también abre `vim` (alias de `/etc/profile.d/vim.sh`). Para editar archivos del sistema: `sudo vim /etc/archivo` o `sudoedit /etc/archivo`. Si vim muestra `[readonly]` o `E45`, se abrió sin `sudo` un archivo de root: `:q!` y repetir con `sudo`.

`nano`: `Ctrl+O` guardar (pide nombre, `Enter`), `Ctrl+X` salir, `Ctrl+W` buscar, `Ctrl+\` reemplazar, `Ctrl+K` cortar línea, `Ctrl+U` pegar, `Alt+U` deshacer, `Ctrl+G` ayuda. Los atajos aparecen abajo en pantalla.

### Lab 6.1 — Editar una configuración con vim (10 min)

- **Objetivo:** abrir, modificar, buscar, reemplazar, guardar y salir sin perder nada.
- **Ritmo:** núcleo = pasos 1–4 (nadie sigue hasta que todos hayan salido de vim con `:wq` y con `:q!`). Los pasos 5 y 6 son demo o tarea.

1. Crear el archivo de prueba.

```bash
cat > ~/empresa/documentos/app.conf <<'CONF'
# Configuracion de la aplicacion PanamaTech
servidor=srv-old
puerto=8080
usuario=ana
ruta_logs=/var/log/app
debug=false
db_host=srv-old
db_puerto=5432
backup_host=srv-old
CONF
vim --version | head -1
```
Salida esperada:
```
VIM - Vi IMproved 8.2 (2019 Dec 12, compiled ...)
```

2. Abrir y practicar. Cada acción en su tecla; el instructor las dicta una a una.

```bash
vim ~/empresa/documentos/app.conf
```
Secuencia:
1. `:set nu` `Enter` → aparecen los números de línea.
2. `/srv-old` `Enter`, `n`, `n` → salta entre las tres coincidencias; `N` vuelve.
3. `:%s/srv-old/srv-new/g` `Enter` → abajo: `3 substitutions on 3 lines`.
4. `G` (última línea), `o` (nueva línea debajo, ya en modo insertar), escribir `# modificado por student`, `Esc`.
5. `gg`, `yy`, `p` → la línea de comentario queda duplicada; `dd` borra la copia; `u` la devuelve; `Ctrl+R` la vuelve a borrar.
6. Ir a la línea 6 con `:6` `Enter`, `$` (fin de línea), `x` borra la `e` final de `false`... y `u` para deshacerlo (solo para ver `u` en acción).
7. `:wq` `Enter`.

3. Verificar desde la shell.

```bash
grep -n "srv-" ~/empresa/documentos/app.conf
tail -1 ~/empresa/documentos/app.conf
```
Salida esperada:
```
2:servidor=srv-new
7:db_host=srv-new
9:backup_host=srv-new
# modificado por student
```

4. Salir sin guardar (el rescate más importante).

```bash
vim ~/empresa/documentos/app.conf
```
Secuencia: `i`, escribir cualquier cosa desordenada, `Esc`, `:q` → vim responde `E37: No write since last change (add ! to override)`; `:q!` `Enter` → sale sin cambios. Verificar:

```bash
head -2 ~/empresa/documentos/app.conf
```
Salida esperada:
```
# Configuracion de la aplicacion PanamaTech
servidor=srv-new
```

5. Un archivo real en modo solo lectura (sin riesgo).

```bash
vim -R /etc/ssh/ssh_config
```
Secuencia: `:set nu`, `/Include`, `n`, `G`, `gg`, `:q`. Después, para ver qué pasa sin permisos:

```bash
vim /etc/hosts
```
Secuencia: `i`, escribir una letra, `Esc`, `:wq` → `E45: 'readonly' option is set (add ! to override)` (o `E212: Can't open file for writing`). `:q!`. Explicar: se abrió sin `sudo`. La forma correcta sería `sudo vim /etc/hosts` o `sudoedit /etc/hosts`. Hoy no se edita.

6. `nano` como alternativa.

```bash
nano ~/empresa/documentos/notas.txt
```
Escribir dos líneas, `Ctrl+O`, `Enter`, `Ctrl+X`. Luego `cat ~/empresa/documentos/notas.txt`.

- **Checkpoint:** pegar en el chat la salida de:

```bash
grep -c srv-new ~/empresa/documentos/app.conf; tail -1 ~/empresa/documentos/app.conf; rpm -q vim-enhanced
```

Salida esperada (la versión exacta del paquete varía con la versión menor de RHEL; en UTM termina en `.aarch64`):
```
3
# modificado por student
vim-enhanced-8.2.2637-20.el9_1.x86_64
```

---

## Reto individual (20 min)

**Ticket #2026-0142 — Análisis de log de acceso y respaldo**

Se recibe del área de seguridad un log de 200 líneas (`~/reto/acceso.log`, mismo formato del lab: `FECHA HORA NIVEL USUARIO SERVICIO MENSAJE`). Se pide:

1. ¿Cuántas líneas contienen `FAIL`?
2. ¿Cuál es la última línea `WARN` del archivo (en orden del archivo)? Bonus: ¿y la más reciente en el tiempo?
3. Crear `~/reto/ana.log` únicamente con las líneas del usuario `ana`, ordenadas cronológicamente (fecha y hora).
4. Empaquetar y comprimir el directorio `~/reto` en `~/backups/reto-AAAA-MM-DD.tar.gz` (la fecha de hoy, generada con `date`, no escrita a mano) y comprobar listando su contenido.
5. Crear un enlace simbólico `~/ultimo-backup` que apunte a ese `.tar.gz`, de modo que `tar -tzf ~/ultimo-backup` funcione desde cualquier directorio.

Bonus: ¿qué servicio acumula más `ERROR` + `FAIL`?

Lo que no se termine en clase queda como tarea.

**Entrega** (pegar en el chat la salida completa):

```bash
grep -cw FAIL ~/reto/acceso.log; grep -w WARN ~/reto/acceso.log | tail -1; wc -l ~/reto/ana.log; ls -l ~/ultimo-backup; tar -tzf ~/ultimo-backup
```

**Generación del log (el instructor pega este bloque en el chat; todos lo ejecutan):**

```bash
mkdir -p ~/reto && cat > ~/reto/genera-reto.sh <<'FIN'
#!/bin/bash
# genera-reto.sh: log de 200 lineas con semilla fija (todos obtienen el mismo archivo)
RANDOM=2026
niveles=(INFO INFO WARN ERROR FAIL)
usuarios=(ana carlos pedro root backup)
servicios=(sshd httpd crond nfs-server backup.sh)
mensajes=("Sesion iniciada" "Sesion cerrada" "Conexion rechazada" "Archivo no encontrado" "Tarea completada" "Tiempo de espera agotado" "Permiso denegado" "Disco al 90 por ciento")
for i in $(seq 1 200); do
  printf "2026-04-%02d %02d:%02d:%02d %s %s %s %s\n" \
    $((RANDOM % 30 + 1)) $((RANDOM % 24)) $((RANDOM % 60)) $((RANDOM % 60)) \
    "${niveles[RANDOM % 5]}" "${usuarios[RANDOM % 5]}" \
    "${servicios[RANDOM % 5]}" "${mensajes[RANDOM % 8]}"
done > ~/reto/acceso.log
FIN
bash ~/reto/genera-reto.sh && wc -l ~/reto/acceso.log && md5sum ~/reto/acceso.log
```

⚠️ Verificar en la VM antes de la clase: el instructor ejecuta lo mismo en su VM la noche anterior, anota el `md5sum` y las respuestas reales (este documento no las trae hechas a propósito), y en clase compara el `md5sum` de cada participante. Si a alguien no le coincide, se le pide que pegue su salida de la entrega y se verifica con los comandos de la solución sobre su propio archivo.

### Solución (para el instructor)

```bash
# 1. Lineas FAIL
grep -cw FAIL ~/reto/acceso.log

# 2. Ultima linea WARN en orden del archivo; y la mas reciente en el tiempo
grep -w WARN ~/reto/acceso.log | tail -1
grep -w WARN ~/reto/acceso.log | sort -k1,2 | tail -1

# 3. Solo lineas de ana (campo 4), ordenadas por fecha y hora
grep -w ana ~/reto/acceso.log | sort -k1,2 > ~/reto/ana.log
#    forma estricta por campo, por si el nombre apareciera en un mensaje:
grep -E "^[0-9-]+ [0-9:]+ [A-Z]+ ana " ~/reto/acceso.log | sort -k1,2 > ~/reto/ana.log
wc -l ~/reto/ana.log
cut -d' ' -f4 ~/reto/ana.log | sort -u        # debe imprimir solo: ana

# 4. Empaquetar con fecha
mkdir -p ~/backups
tar -czf ~/backups/reto-$(date +%F).tar.gz -C ~ reto
tar -tzf ~/backups/reto-$(date +%F).tar.gz

# 5. Enlace simbolico con ruta absoluta
ln -s /home/student/backups/reto-$(date +%F).tar.gz ~/ultimo-backup
ls -l ~/ultimo-backup
cd /tmp && tar -tzf ~/ultimo-backup && cd

# Bonus: servicio con mas ERROR+FAIL
grep -Ew "ERROR|FAIL" ~/reto/acceso.log | cut -d' ' -f5 | sort | uniq -c | sort -rn | head -1
```

Errores que se verán: enlace creado con ruta relativa desde otro directorio (roto: `ls -l` en rojo, `tar` dice `No such file`); `sort` sin `-k1,2` (funciona igual porque la fecha va primero: aceptar y explicar); `grep ana` sin `-w` (aceptar si el resultado es igual, comentar el riesgo); fecha escrita a mano en el nombre del tar (pedir que repitan con `$(date +%F)`).

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `pwd` / `cd ruta` / `cd -` / `cd` | Dónde estoy / ir / volver al anterior / ir al home |
| `ls -lah`, `ls -lt`, `ls -lS`, `ls -R`, `ls -ld dir`, `ls -li` | Listar con detalle, por fecha, por tamaño, recursivo, el directorio en sí, con inodo |
| `echo $PATH`, `env`, `export VAR=valor`, `echo $?` | Ver variables, exportarlas, código de salida del último comando |
| `alias lh='ls -lh'`, `unalias lh`, `type cmd` | Crear/quitar alias; saber si algo es alias, builtin o programa |
| `history`, `!!`, `!n`, `!$`, `Ctrl+R` | Historial y repetición de comandos |
| `man cmd`, `man -k texto`, `cmd --help`, `man 5 archivo` | Documentación |
| `mkdir -p a/b/c` | Crear directorios anidados |
| `touch archivo`, `touch -d "2020-01-15" archivo` | Crear vacío / fijar fecha |
| `cp -r`, `cp -a`, `cp -iv` | Copiar recursivo / conservando todo / preguntando y contando |
| `mv -i origen destino` | Mover o renombrar |
| `rm -i`, `rm -r`, `rmdir` | Borrar (preguntando), recursivo, directorio vacío. Nunca `rm -rf` sin `ls` previo |
| `ln archivo duro`, `ln -s /ruta/absoluta enlace` | Enlace duro / simbólico |
| `stat archivo`, `stat -c '%y %n'`, `file archivo` | Metadatos completos / con formato / tipo real de contenido |
| `tree dir` | Árbol de directorios |
| `cat -n`, `less`, `head -n 5`, `tail -n 5`, `tail -f` | Ver texto; seguir un log en vivo |
| `wc -l`, `nl`, `tr 'a-z' 'A-Z'`, `diff -r a b` | Contar, numerar, transformar, comparar |
| `sort -n -r -u -k2 -t:`, `uniq -c` | Ordenar; contar repetidos (`sort \| uniq -c \| sort -rn`) |
| `cut -d' ' -f3`, `cut -d: -f1,7` | Extraer campos |
| `grep -i -v -n -c -w -r -l -E "A\|B"` | Filtrar texto con expresiones regulares |
| `cmd > f`, `>> f`, `2> f`, `> f 2>&1`, `&> f`, `2>/dev/null`, `< f` | Redirección de salida, error y entrada |
| `cmd \| tee -a archivo`, `echo x \| sudo tee /etc/f` | Guardar y mostrar; escribir en archivos de root |
| `cmd \| xargs -n1 otro` | Convertir salida en argumentos |
| `find /ruta -name "*.conf" -type f -size +1M -mtime -1 -user u -perm 000 -empty -newer f` | Buscar por criterios |
| `find ... -exec cmd {} \;` / `-exec cmd {} +` / `-ok` / `-delete` | Actuar sobre lo encontrado |
| `sudo updatedb`, `locate -i nombre` | Búsqueda instantánea en base de datos |
| `which cmd`, `whereis cmd`, `type cmd` | Dónde está un programa |
| `tar -czf salida.tar.gz -C /base dir --exclude='*.log'` | Crear respaldo comprimido |
| `tar -tzf archivo`, `tar -tvzf archivo` | Listar contenido |
| `tar -xzf archivo -C /destino [ruta/dentro]` | Restaurar (todo o un archivo) |
| `gzip -k f`, `bzip2 -k f`, `xz -k f`, `zcat f.gz`, `gunzip` | Comprimir/descomprimir archivos sueltos |
| `zip -r a.zip dir`, `unzip -l a.zip`, `unzip a.zip -d dir` | Formato zip |
| `du -sh dir`, `du -sh dir/* \| sort -h`, `ls -lh` | Cuánto ocupa |
| `vim archivo`: `i` `Esc` `:wq` `:q!` `dd` `yy` `p` `u` `/texto` `:set nu` `:%s/a/b/g` | Editar |
| `nano archivo`: `Ctrl+O` `Ctrl+X` `Ctrl+W` | Editor alternativo |

---

## Notas para el instructor

### Preparar antes de la clase
- Restaurar el snapshot `dia01-fin` en la VM del instructor y recorrer **todos** los labs de este documento de principio a fin (unos 60 min). Anotar el `md5sum` de `servidor.log` (semilla 124) y de `acceso.log` (semilla 2026) y las respuestas del reto en la propia VM; los valores de este documento son ilustrativos.
- Verificar en la VM: `sudo dnf install -y tree vim-enhanced bash-completion zip unzip bzip2 nano mlocate` funciona con la suscripción de desarrollador. En RHEL 9 el paquete es `mlocate` (BaseOS); `plocate` es el reemplazo en RHEL 10. ⚠️ Verificar en la VM antes de la clase: si `mlocate` no apareciera, probar `plocate` y ajustar el Lab 4.1.
- Tener el bloque del generador del reto y el de `genera-log.sh` en un archivo de texto listo para pegar en el chat (Teams/Zoom rompen a veces las líneas largas: pegarlos como bloque de código o compartir un enlace a un `.txt`).
- Tener abierta una segunda sesión SSH a la propia VM para demostrar `tail -f` con escritura desde otra terminal.
- Preparar la demostración de `rm -rf ~/empresa /backups` (escribir sin ejecutar) y la de `sudo echo > /etc/motd` que falla.
- Tener el enlace a la OVA/imagen de respaldo por si alguien llega sin VM funcional; y recordar el procedimiento de snapshot del Día 1 (VirtualBox: Máquina → Instantáneas; UTM: con la VM apagada, según lo explicado el Día 1).
- Comprobar cómo se escriben `|`, `~`, `{`, `}` y `\` en el teclado de cada participante (preguntarlo al inicio; ver más abajo).

### Qué estudiar si es nuevo en RHEL
1. **UsrMove y `/etc/profile.d`.** Ejecutar `ls -l /` y `ls /etc/profile.d/` y leer `colorls.sh`, `which2.sh`, `vim.sh` y `/etc/bashrc`: explican por qué `ls`, `ll`, `grep`, `which` y `vi` se comportan como alias en RHEL. Saberlo evita quedarse sin respuesta cuando alguien pregunte "¿por qué `which ls` me muestra un alias?".
2. **El punto de SELinux en `ls -l` y `stat`.** `ls -lZ /etc/hostname`, `man ls` (opción `-Z`). Hoy solo hay que poder decir "es el contexto de SELinux, lo vemos el Día 8", pero conviene haberlo visto.
3. **`mlocate` vs `plocate` en RHEL 9.** `dnf search locate`, `dnf info mlocate`, `rpm -ql mlocate | grep -E "timer|service"`, `man updatedb.conf` (rutas excluidas, `PRUNEPATHS`).
4. **`find` a fondo.** `man find`: secciones TESTS y ACTIONS, en particular la diferencia `-exec {} \;` vs `+`, `-perm` con `-` y `/`, y el orden de `-delete`. Practicar `find /etc -mmin -60` después de instalar un paquete para ver qué toca dnf.
5. **`tar` y ownership.** `man tar`: `--same-owner` (por defecto solo para root), `-p`, `--exclude`, `-C`, `-a`. Probar la restauración de `/etc` con y sin `sudo` y comparar dueños con `ls -l`.
6. **`vim-minimal` vs `vim-enhanced` y `sudoedit`.** `rpm -q vim-minimal vim-enhanced`, `man sudoedit`, `echo $EDITOR`. En RHEL 9 con `nano` instalado, `EDITOR` puede apuntar a nano (`/etc/profile.d/nano-default-editor.sh`): saberlo evita sorpresas con `crontab -e` y `visudo` más adelante.
7. **`systemd-tmpfiles`.** `cat /usr/lib/tmpfiles.d/tmp.conf`, `man 5 tmpfiles.d`: por qué `/tmp` se limpia a los 10 días y `/var/tmp` a los 30 (recordarlo cuando alguien deje un respaldo en `/tmp`).
8. **Historia y readline.** `man bash`, secciones HISTORY EXPANSION y READLINE, y `man 7 glob`.

### Mini-plan de 5 minutos para introducir vim sin asustar
1. (30 s) "vim tiene dos estados: *escribir* y *mandar*. Se entra a escribir con `i` y se vuelve a mandar con `Esc`. Cuando no sepan en qué estado están: `Esc` dos veces."
2. (30 s) "Hoy solo necesitan seis teclas: `i`, `Esc`, `:wq`, `:q!`, `dd`, `u`. El resto es velocidad, no necesidad."
3. (60 s) Demostración en vivo: abrir un archivo nuevo, `i`, escribir dos líneas, `Esc`, `:wq`. Reabrir, `dd`, `u`, `:q!`.
4. (90 s) Todos lo repiten con `vim ~/prueba.txt`: escribir su nombre, guardar, salir; reabrir, romper algo, `:q!`. Nadie sigue hasta que todos han salido de vim.
5. (60 s) Presentar `/`, `n`, `:set nu`, `:%s/a/b/g` como "atajos que ahorran tiempo en el examen", y mandar `vimtutor` (20 min) como tarea. Cerrar con: "en el examen RHCSA hay `vim` seguro; `nano` puede estar o no. Si algo se descontrola, `Esc` `:q!` y se vuelve a empezar; el archivo no cambia hasta que hagan `:w`".

### Advertencia sobre `rm -rf` (decirla completa)
- No hay papelera ni "deshacer". Lo borrado se recupera solo desde un respaldo o un snapshot.
- Antes de un `rm -r`, ejecutar `ls` con la misma ruta y mirar el resultado. Con patrones, `echo patrón` antes.
- Nunca `rm -rf` con variables sin comprobar: `rm -rf "$DIR"/*` con `DIR` vacío se expande a `rm -rf /*` y sí borra el sistema (`--preserve-root`, que está activo por defecto, solo bloquea el argumento `/` escrito tal cual). En scripts, verificar que la variable no esté vacía (Día 7).
- Cuidado con el espacio: `rm -rf ~/empresa /backups` son dos argumentos.
- Como root, RHEL ya define `alias rm='rm -i'`; como `student` conviene hacer lo mismo en `~/.bashrc` mientras se aprende.
- Recordar que `sudo rm -rf` puede borrar cualquier cosa, incluidos los archivos del sistema; `--preserve-root` solo protege `/` exactamente.

### Diferencias de teclado en SSH y consola
- Por SSH, las teclas las interpreta la terminal del **equipo del participante**; la distribución configurada dentro de la VM (`localectl status`) solo afecta a la consola de VirtualBox/UTM. Si en la consola aparecen caracteres cambiados: `sudo localectl set-keymap la-latin1` (o `es`, según el teclado físico).
- Los caracteres problemáticos son `|`, `~`, `{`, `}`, `\` y `>`. En teclados latinoamericanos y españoles suelen requerir `AltGr`; en Mac, `Option` (por ejemplo, en Mac con teclado español: `|` = Option+1, `~` = Option+ñ; verificar en cada equipo). Pedir al inicio de la clase que cada participante escriba `echo "|~{}\\"` para comprobarlo.
- Pegar en la terminal: Windows Terminal/PowerShell `Ctrl+Shift+V` o clic derecho; PuTTY clic derecho o `Shift+Insert`; Mac `Cmd+V`. `Ctrl+V` dentro de vim en modo insertar inserta caracteres literales, no pega.
- En Mac Terminal, `Option` no actúa como `Alt`/Meta por defecto (Preferencias → Perfiles → Teclado → "Usar Opción como tecla Meta"); solo importa para atajos como `Alt+.`.
- `Ctrl+S` congela la terminal (control de flujo): se ve como "se trabó SSH". `Ctrl+Q` la libera. Es el problema más frecuente del día.
- Si `Backspace` imprime `^?` o `^H` en algún cliente: `stty erase ^?` (o `^H`) en la VM lo corrige para esa sesión.

### Si alguien no tiene SSH funcionando
1. Que trabaje desde la **consola de la VM** (ventana de VirtualBox/UTM) mientras se sigue la clase: todo el material funciona igual, salvo el copiar/pegar (no hay) y el scroll (`Shift+PgUp`/`PgDn`, limitado). Que teclee los comandos en lugar de pegarlos; el instructor los dicta despacio.
2. Diagnosticar en el descanso, en este orden: (a) en la VM, `systemctl status sshd` debe decir `active (running)`; si no, `sudo systemctl enable --now sshd`; (b) `sudo ss -tlnp | grep :22` debe mostrar `0.0.0.0:22` (y `[::]:22`); (c) regla de port forwarding en el hipervisor: protocolo TCP, host 2222, invitado 22 (en VirtualBox, con la VM en red NAT; en UTM, "Emulated VLAN" con port forward); (d) desde el equipo propio, `ssh -p 2222 student@127.0.0.1` (a veces `localhost` resuelve a IPv6 y falla); (e) firewall o antivirus del equipo propio bloqueando el cliente SSH.
3. Si no se resuelve, ofrecer la OVA de respaldo para restaurar en casa y seguir el resto del día por consola.

### Errores frecuentes de los participantes y cómo resolverlos

| Síntoma | Causa | Solución |
|---|---|---|
| `Permission denied` al leer `/var/log/messages` o `/etc/shadow` | Archivos de root | Anteponer `sudo` |
| `-bash: /etc/motd: Permission denied` con `sudo echo ... > /etc/motd` | La redirección la hace la shell del usuario, no `sudo` | `echo ... \| sudo tee /etc/motd` |
| `bash: !": event not found` | `!` dentro de comillas dobles | Comillas simples, o `set +H` |
| `tar: Removing leading '/' from member names` | Aviso, no error | Ignorar; explica por qué se restaura en `/tmp/restaurar/etc` |
| `find: '/etc/...': Permission denied` | Directorios no legibles | `sudo find` o `2>/dev/null` |
| `locate: can not stat () ... mlocate.db` | Base de datos no creada | `sudo updatedb` |
| `locate` no encuentra un archivo recién creado | La base es del último `updatedb` | `sudo updatedb` y repetir |
| `rm: cannot remove 'dir': Is a directory` | Falta `-r` | `rm -r dir` (con `ls` previo) |
| `rmdir: Directory not empty` | `rmdir` solo borra vacíos | `rm -r` o vaciar antes |
| `mkdir: cannot create directory 'a/b': No such file or directory` | Falta el padre | `mkdir -p a/b` |
| `cp -r dir destino` dejó `destino/dir` en vez de `destino` | `destino` ya existía | Borrar y repetir, o `cp -r dir/. destino/` |
| `uniq -c` da cuentas sueltas repetidas | Entrada sin ordenar | `sort` antes de `uniq` |
| `sort` deja `1000` antes de `65534` | Orden alfabético | `sort -n` |
| `grep ana` incluye líneas que no son del usuario | Coincidencia dentro de palabras | `grep -w ana` o filtrar por campo con `cut` |
| Atrapado en `vim`, la pantalla no responde a `:q` | Está en modo insertar | `Esc` y luego `:q!` |
| `E37: No write since last change` | Cambios sin guardar | `:wq` para guardar o `:q!` para descartar |
| `E45: 'readonly' option is set` / `E212` | Archivo de root abierto sin `sudo` | `:q!` y `sudo vim archivo` |
| La terminal "se trabó" | `Ctrl+S` | `Ctrl+Q` |
| Desapareció vim o less y volvió el prompt | `Ctrl+Z` suspendió el proceso | `fg` |
| Se cerró la sesión SSH de golpe | `Ctrl+D` en un prompt vacío | Volver a conectar |
| `tree: command not found` / `vim: command not found` | Paquete no instalado | `sudo dnf install -y tree vim-enhanced` |
| `dnf`: `This system is not registered` / `no repositories` | Falta registro | `sudo subscription-manager register` (Día 1) |
| El enlace simbólico sale en rojo y `cat` dice `No such file` | Creado con ruta relativa desde otro directorio, o destino borrado | Recrearlo con ruta absoluta: `ln -sf /ruta/absoluta enlace` |
| `tar -tzf` dice `not in gzip format` | Se usó `-z` sobre un `.tar`, `.bz2` o `.xz` | Quitar la letra de compresión o usar la correcta (`-j`, `-J`); `tar -tf` la detecta solo |
| `md5sum` del log no coincide con el del instructor | Versión de bash distinta o generador editado | No importa: usar los mismos comandos; las cifras cambian |
| `2>&1 > archivo` sigue mostrando errores en pantalla | Orden de redirección | `> archivo 2>&1` o `&> archivo` |
| `systemctl sta<Tab>` no completa subcomandos | Falta `bash-completion` o la sesión se abrió antes de instalarlo | `sudo dnf install -y bash-completion` y abrir una sesión SSH nueva |
| `tar: Cowardly refusing to create an empty archive` | Se olvidó decir qué empaquetar: `tar -czf empresa.tar.gz` sin la ruta | Añadir qué se empaqueta: `tar -czf empresa.tar.gz -C ~ empresa` |
| Se creó un archivo llamado `z` (o `f`) en vez del `.tar.gz` | Las letras se pusieron en otro orden: `tar -cfz x.tar.gz dir` toma `z` como nombre del archivo | La `f` va **última** del grupo de letras, porque le sigue el nombre: `tar -czf x.tar.gz dir` |
| `ls -l` sobre la carpeta restaurada muestra dueño `student` en archivos de `/etc` | Se extrajo sin `sudo`: `tar` solo conserva dueños si es root | `sudo tar -xzf ...` |

### Diferencias VirtualBox (x86_64) vs UTM (aarch64) en este día
- `uname -m`: `x86_64` vs `aarch64`. `file /usr/bin/ls` dice `x86-64` vs `ARM aarch64`; `interpreter /lib64/ld-linux-x86-64.so.2` vs `/lib/ld-linux-aarch64.so.1`.
- Discos en `/dev`: `sda`, `sda1`, `sda2` vs `vda`, `vda1`, `vda2` (números mayor/menor `8,0` vs `252,0`). Por eso el lab usa `ls /dev/sd* /dev/vd* 2>/dev/null`.
- Interfaz en `/sys/class/net`: `enp0s3` vs `enp0s1` (aprox.; verificar con `nmcli device`).
- `/boot/efi` existe en UTM (UEFI); en VirtualBox solo si se activó EFI al crear la VM.
- `/proc/cpuinfo` en aarch64 no tiene `model name`; usar `lscpu` o `grep -c processor`.
- Los nombres de paquete terminan en `.x86_64` vs `.aarch64` (`rpm -q vim-enhanced`).
- Tamaños de `tar`/`gzip` casi idénticos; los binarios de `/usr/bin` difieren ligeramente en tamaño (`ls -l /bin/bash`).

### Preguntas probables y respuesta corta
- **¿Por qué `/bin` es un enlace a `/usr/bin`?** Desde RHEL 7 todos los programas viven en `/usr` ("UsrMove"); los enlaces mantienen compatibilidad con rutas antiguas como `#!/bin/bash`.
- **¿Qué es el punto después de los permisos?** Indica que el archivo tiene contexto SELinux; `ls -Z` lo muestra. Se ve el Día 8.
- **¿Cuándo uso enlace duro y cuándo simbólico?** Casi siempre simbólico: se ve a dónde apunta, puede cruzar discos y apuntar a directorios. El duro sirve para que un archivo sobreviva al borrado de un nombre, dentro del mismo filesystem.
- **¿`rm` borra de verdad? ¿Se puede recuperar?** Sí borra; no hay papelera. Recuperación solo desde respaldo o snapshot. Por eso hoy aprendimos `tar`.
- **¿Por qué `sudo echo texto > archivo` falla?** La redirección la ejecuta la shell de `student` antes de que `sudo` intervenga. Usar `| sudo tee`.
- **¿`tar.gz` o `zip`?** Dentro de Linux, `tar.gz` (conserva permisos y dueños). Para enviar a usuarios de Windows, `zip`.
- **¿`tar` conserva permisos y dueños?** Permisos siempre; dueños al restaurar solo si se extrae como root.
- **¿`locate` o `find`?** `locate` para "¿dónde está el archivo X?" (instantáneo, puede estar desactualizado); `find` para criterios (tamaño, fecha, dueño) y para actuar sobre los resultados.
- **¿Por qué vim y no nano?** Está en todos los sistemas, incluso en modo rescate, y es lo que garantiza el examen RHCSA. nano es válido para el trabajo diario si está instalado.
- **¿`grep -E` o `egrep`?** `grep -E`; `egrep` es un alias antiguo que se está retirando.
- **¿Qué diferencia hay entre `>` y `>>`?** `>` sobrescribe; `>>` añade al final. Con `set -o noclobber`, `>` no sobrescribe sin `>|`.
- **¿Puedo hacer que `ls -l` muestre fechas completas?** `ls -l --time-style=long-iso` o `ls -l --full-time`.
- **¿Por qué `sort` ordena "raro" las mayúsculas y los acentos?** Depende de `LANG`; `LANG=C sort` ordena por código de carácter.

### Relación con el examen RHCSA (EX200)
Este día cubre casi todo el objetivo **"Understand and use essential tools"**:
- Access a shell prompt and issue commands with correct syntax.
- Use input-output redirection (`>`, `>>`, `|`, `2>`, etc.).
- Use grep and regular expressions to analyze text.
- Create, delete, copy, and move files and directories.
- Create hard and soft links.
- Archive, compress, unpack, and uncompress files using tar, gzip, and bzip2.
- Create and edit text files (vim).
- Locate, read, and use system documentation including man, info, and files in `/usr/share/doc`.
Quedan para otros días: acceder por SSH (Día 5), cambiar de usuario (Día 3), permisos ugo/rwx (Día 3). Consejo de examen que hay que transmitir: en el EX200 se pierde más tiempo por no saber salir de vim o por un `tar` mal escrito que por desconocer un servicio; la fluidez de hoy es puntos directos.

---

## Tarea y preparación para el día siguiente

1. **Snapshot** `dia02-fin` con la VM apagada (`sudo poweroff`), como se hizo el Día 1. Si algo quedó roto, restaurar `dia01-fin` y volver a ejecutar los labs 2.1 y 3.1 (15 min).
2. **Terminar el reto** si quedó pendiente y guardar la salida de la entrega.
3. **`vimtutor`** (20 min): `vimtutor` en la VM (o `vimtutor es` para la versión en español, si está disponible). Completar al menos las lecciones 1 a 4.
4. **Práctica de 20 minutos** (sin mirar el material):
   - Crear `~/practica/{a,b,c}` con un archivo en cada uno, empaquetarlos en `/tmp/practica.tar.gz` y restaurarlos en `/tmp/r`.
   - `sudo find /var/log -mmin -60` y explicar en una línea qué archivos aparecen y por qué.
   - Extraer de `/etc/passwd` los usuarios con shell `/bin/bash` (`grep`, `cut`), ordenados.
   - Abrir `~/empresa/documentos/app.conf` con vim, cambiar `puerto=8080` por `puerto=9090` con `:%s`, guardar.
   - `grep -Ev "^#|^$" /etc/ssh/ssh_config` y contar las líneas.
5. **Lectura** (10 min): `man 7 hier` y `man tar` (sección de ejemplos al inicio).
6. **Día 3 (Usuarios, grupos, permisos y sudo):** no requiere hardware ni paquetes adicionales. Se construirá la estructura institucional (usuarios `ana`, `carlos`, `pedro`, `laura`; grupos `sistemas`, `soporte`, `auditoria`; carpetas compartidas `/srv/sistemas`, `/srv/soporte`, `/srv/publico`). Conviene llegar con `~/empresa` intacto: se usará para practicar `chmod`/`chown` sobre archivos propios. Repasar de hoy: lectura de `ls -l`, `find -perm`, `stat`, y `vi` básico (`visudo` abre `vi`).
