# Día 03 — Usuarios, grupos, sudo y permisos
> Al terminar el día, el participante crea y administra usuarios y grupos con política de contraseñas, delega privilegios con `sudo` de forma limitada y auditable, y controla el acceso a archivos y carpetas compartidas con permisos estándar, especiales (setuid, setgid, sticky) y ACL.

**Ficha técnica cubierta:** RH124 M3: Creación de usuarios, Administración de grupos, Permisos estándar y especiales (módulo completo). Complemento RHCSA (EX200): listas de control de acceso (ACL) y configuración de acceso de superusuario (`sudo`). Adelanta de RH124 M4 (Logs del sistema) la lectura de `/var/log/secure` y `journalctl` para auditar el uso de `sudo`.

**Requisitos previos:**
- Snapshot `dia02-fin` tomado; VM `rhel01` encendida y accesible con `ssh -p 2222 student@localhost`.
- `student` pertenece al grupo `wheel` (se marcó "Make this user administrator" en el instalador) y la contraseña de `root` es conocida (se necesita si alguien rompe `sudoers`).
- No se instala software hoy: no hace falta suscripción activa. Los paquetes necesarios ya vienen en la instalación "Server": `shadow-utils`, `passwd`, `sudo`, `acl`. Comprobar con `rpm -q shadow-utils passwd sudo acl`. Si en alguna VM faltara `acl` (poco probable: lo arrastra `systemd`), instalarlo **antes** de la clase con `sudo dnf install -y acl`, que sí requiere suscripción.
- Editor: los participantes ya conocen `vi` básico y `nano` (Día 02). `visudo` abre `vi` por defecto.

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:10 | 10 | Repaso | Tres preguntas del Día 02 con la terminal abierta; objetivos del día |
| 0:10–0:25 | 15 | Bloque 1 — Conceptos | Modelo de usuarios: root vs usuario normal, UID/GID, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`, grupo primario vs suplementario |
| 0:25–0:45 | 20 | Lab 1.1 | Radiografía de las cuentas: `id`, `getent`, `who`, `w`, `last`, lectura de `shadow` |
| 0:45–1:00 | 15 | Bloque 2 — Conceptos | Ciclo de vida de una cuenta: `useradd` y sus defaults, `passwd`, `usermod`, `chage`, bloqueo, `userdel`, grupos y `gpasswd` |
| 1:00–1:35 | 35 | Lab 2.1 y 2.2 | Estructura institucional (grupos `sistemas`, `soporte`, `auditoria`; usuarios ana, carlos, pedro, laura); política de contraseñas y bloqueo de cuentas |
| 1:35–1:50 | 15 | Descanso | |
| 1:50–2:00 | 10 | Bloque 3 — Conceptos | `su` vs `su -`, `sudo`, `/etc/sudoers`, `visudo`, `/etc/sudoers.d/`, grupo `wheel`, reglas por comando, auditoría |
| 2:00–2:20 | 20 | Lab 3.1 | Delegación limitada con `sudo` para el grupo `soporte`; verificación con `sudo -l`; rastro en `/var/log/secure` |
| 2:20–2:40 | 20 | Bloque 4 — Conceptos | Permisos rwx, octal y simbólico, permisos en directorios, `umask`, setuid/setgid/sticky, ACL |
| 2:40–2:55 | 15 | Lab 4.1 | Permisos estándar y `umask` en `~/permisos-lab` |
| 2:55–3:20 | 25 | Lab 4.2 | Carpetas compartidas `/srv/sistemas`, `/srv/soporte`, `/srv/publico`: setgid, sticky, ACL para auditoría |
| 3:20–3:50 | 30 | Reto individual | Ticket con cuatro requisitos y verificación |
| 3:50–4:00 | 10 | Cierre | Resumen, cheatsheet, snapshot `dia03-fin`, tarea |

## Prioridad si falta tiempo

**Imprescindible**
- Leer `/etc/passwd` y `/etc/shadow` campo por campo; saber qué significan `!!`, `!` y `$6$`.
- Crear grupos y usuarios con `groupadd`, `useradd -m -c -G`, `passwd`; agregar a grupos con `usermod -aG` (y saber por qué el `-a` es obligatorio).
- `chage -l`, `chage -M 90 -W 7`, `chage -d 0`; bloquear/desbloquear con `passwd -l/-u`.
- Leer `ls -l`, `chmod` octal y simbólico, `chown user:group`.
- Carpeta de grupo con setgid (`chmod 2770`) y demostración de acceso permitido/denegado con `sudo su -`.
- Regla `sudo` limitada en `/etc/sudoers.d/` creada con `visudo -f` y verificada con `sudo -l`.

**Importante**
- `umask` (comprobar el valor real de cada usuario, no asumirlo).
- Sticky bit en `/srv/publico`; setuid en `/usr/bin/passwd` y búsqueda con `find -perm -4000`.
- ACL básicas: `setfacl -m g:auditoria:rX`, `getfacl`, el `+` en `ls -l`.
- Auditoría de `sudo` en `/var/log/secure` y `journalctl -t sudo`.
- `userdel -r` y archivos huérfanos (`find -nouser`).

**Si sobra tiempo** (los pasos correspondientes están marcados como *Opcional* dentro de cada lab)
- ACL de usuario suelto y su límite en el directorio (Lab 4.2, paso 8); el efecto de `chmod` sobre la `mask` (se comenta, no se practica).
- `gpasswd -A` (administrador de grupo) y `newgrp` (Lab 2.1, pasos 8 y 10).
- `PASS_MAX_DAYS` en `/etc/login.defs` para usuarios futuros.
- Script `crear-usuarios.sh` (Lab 2.3): puede quedar como demo del instructor; se retoma el Día 07.

## Repaso del Día 02 (10 min)

Tres preguntas rápidas, con la terminal abierta; un participante distinto responde cada una ejecutando el comando:

1. ¿Cómo encuentro todos los archivos `.log` dentro de `~/empresa`? (`find ~/empresa -name "*.log"`)
2. ¿Cómo cuento cuántas líneas contienen `ERROR` en `~/empresa/logs/servidor.log`? (`grep -c ERROR ~/empresa/logs/servidor.log`)
3. ¿Cómo empaqueto y comprimo `~/empresa` en un solo archivo y cómo veo su contenido sin extraer? (`tar -czf empresa.tar.gz -C ~ empresa` y `tar -tzf empresa.tar.gz`)

Cierre del repaso: hasta hoy todo se hizo como `student`. Hoy la pregunta cambia de "cómo hago X" a "**quién** puede hacer X y cómo lo controlo".

## Bloque 1 — El modelo de usuarios de Linux

### Conceptos (15 min)

**Para el kernel no existen nombres, existen números.** Cada proceso corre con un UID (identificador de usuario) y uno o varios GID (identificadores de grupo). Los nombres `student`, `root` o `ana` son traducciones que hacen las herramientas consultando `/etc/passwd`. Analogía: el UID es el número de cédula; el nombre es cómo lo llaman. Si dos cuentas tienen el mismo UID, para el sistema son la misma persona.

**root es UID 0.** No es especial por llamarse `root`, sino por tener UID 0: el kernel no le aplica las verificaciones de permisos rwx. Cualquier otra cuenta con UID 0 también sería root (eso es exactamente lo que busca un atacante y por lo que se auditan `/etc/passwd` y los binarios setuid).

**Rangos de UID en RHEL 9** (definidos en `/etc/login.defs`):
- `0`: root.
- `1–200`: reservados para cuentas de sistema asignadas de forma fija por paquetes (bin, daemon, sshd, chrony...). La lista oficial está en `/usr/share/doc/setup/uidgid`.
- `201–999`: cuentas de sistema asignadas dinámicamente (`useradd -r`). Sin contraseña, sin shell interactiva, normalmente sin home.
- `1000–60000`: usuarios normales. `student` es 1000 porque fue el primero creado en la instalación.
- `65534`: `nobody`, usado por servicios que necesitan "no ser nadie" (NFS, por ejemplo).

**`/etc/passwd`: 7 campos separados por `:`**

```text
ana:x:2001:2001:Ana Rodriguez - Sistemas:/home/ana:/bin/bash
 1  2   3    4            5                  6         7
```
1. login; 2. `x` = "la contraseña está en shadow"; 3. UID; 4. GID del **grupo primario**; 5. GECOS (comentario: nombre completo, departamento); 6. directorio home; 7. shell de inicio de sesión (`/sbin/nologin` = no hay sesión interactiva).

Es legible por todos (`644`) a propósito: `ls -l` necesita traducir UID a nombre sin ser root. Por eso las contraseñas se movieron a otro archivo hace décadas.

**`/etc/shadow`: 9 campos, solo root lo lee (modo `000`)**

```text
ana:$6$Zy9k...:20699:1:90:7::20819:
 1      2        3   4 5  6 7   8   9
```
1. login; 2. hash de la contraseña; 3. fecha del último cambio, en **días desde 1970-01-01**; 4. días mínimos entre cambios; 5. días máximos de validez; 6. días de aviso antes de expirar; 7. días de inactividad tras expirar antes de bloquear; 8. fecha de expiración de la **cuenta** (días desde 1970); 9. reservado.

Sobre el campo 2:
- `$6$sal$hash`: SHA-512, el estándar en RHEL 9 (`ENCRYPT_METHOD SHA512` en `login.defs`). En sistemas más nuevos puede verse `$y$` (yescrypt).
- `!!`: la cuenta no tiene contraseña asignada todavía (así queda tras `useradd`). Nadie puede entrar con contraseña.
- `!` o `!!` **delante** de un hash: contraseña bloqueada (`passwd -l`, `usermod -L`). El hash sigue ahí; al desbloquear se recupera.
- `*`: cuenta de sistema, nunca tendrá contraseña.

**`/etc/group` y `/etc/gshadow`**

```text
sistemas:x:3001:ana,carlos        # nombre:x:GID:miembros suplementarios
sistemas:!:ana:ana,carlos         # gshadow: nombre:contraseña:administradores:miembros
```

**Grupo primario vs suplementarios.** El primario es el GID del campo 4 de `passwd`: es el grupo con el que nacen los archivos que el usuario crea. Los suplementarios son los que aparecen en `/etc/group`: dan acceso adicional. RHEL crea automáticamente un **grupo privado** con el mismo nombre del usuario (`USERGROUPS_ENAB yes`), de modo que `ana` tiene grupo primario `ana` (GID 2001) y grupo suplementario `sistemas`. Esta decisión de diseño es la que permite el `umask 002` que veremos en el Bloque 4.

**`getent` en lugar de `grep`.** `getent passwd ana` consulta a través de NSS (`/etc/nsswitch.conf`): funciona igual si los usuarios vienen de archivos locales, de LDAP o de Active Directory vía `sssd`. `grep ana /etc/passwd` solo ve los locales. En una institución que integre el directorio corporativo, la diferencia importa.

**Quién está aquí y quién estuvo:** `whoami` (yo), `id` (mis UID/GID/grupos), `who` y `w` (sesiones activas; `w` añade qué están ejecutando), `last` (historial de sesiones, de `/var/log/wtmp`), `lastb` (intentos fallidos, `/var/log/btmp`, solo root), `lastlog` (último ingreso de cada cuenta).

### Lab 1.1 — Radiografía de las cuentas (20 min)

- **Objetivo:** leer e interpretar las cuatro bases de datos de cuentas y las herramientas de identidad, sin modificar nada.

1. Quién soy y con qué identidad corre mi shell:

```bash
whoami
id
```
Salida esperada:
```text
student
uid=1000(student) gid=1000(student) groups=1000(student),10(wheel) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```
Qué observar: UID 1000 (primer usuario normal), grupo primario `student` (grupo privado), suplementario `wheel` (GID 10, el grupo de administradores de RHEL). El `context=` es SELinux; se explica el Día 08.

2. La misma información vía NSS y directamente del archivo:

```bash
getent passwd student root
head -3 /etc/passwd
awk -F: '$3 >= 1000 {print $1, $3, $7}' /etc/passwd
```
Salida esperada:
```text
student:x:1000:1000:student:/home/student:/bin/bash
root:x:0:0:root:/root:/bin/bash
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
nobody 65534 /sbin/nologin
student 1000 /bin/bash
```
Qué observar: casi todas las cuentas del sistema usan `/sbin/nologin`. El campo 5 de `student` contiene lo que se escribió como "Full name" en el instalador (puede variar).

3. Los cuatro archivos y sus permisos:

```bash
ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow
cat /etc/shadow
```
Salida esperada:
```text
-rw-r--r--. 1 root root 1301 Sep  3 09:10 /etc/group
----------. 1 root root 1064 Sep  3 09:10 /etc/gshadow
-rw-r--r--. 1 root root 2891 Sep  3 09:10 /etc/passwd
----------. 1 root root 1499 Sep  3 09:10 /etc/shadow
cat: /etc/shadow: Permission denied
```
Qué observar: `shadow` y `gshadow` tienen modo `000`. Ni siquiera root tiene "permiso" rwx: root los lee porque el kernel no le aplica esas verificaciones. El punto final (`.`) tras los permisos indica que el archivo tiene contexto SELinux (todos lo tienen en RHEL).

4. Leer `shadow` como root e interpretar la fecha:

```bash
sudo grep '^student' /etc/shadow
sudo grep '^student' /etc/shadow | cut -d: -f3
date -d "1970-01-01 +$(sudo grep '^student' /etc/shadow | cut -d: -f3) days" +%F
```
Salida esperada:
```text
student:$6$rQm7...(hash largo)...:20699:0:99999:7:::
20699
2026-09-03
```
Qué observar: `0:99999:7` = puede cambiar la contraseña en cualquier momento, nunca expira, avisaría 7 días antes. Los campos 7, 8 y 9 vacíos: sin inactividad ni expiración de cuenta. Ese es el estado por defecto de RHEL: sin política. Lo corregiremos en el Lab 2.2.

5. Grupos:

```bash
getent group wheel student
sudo head -3 /etc/gshadow
grep -E '^(UID|GID|SYS_UID|SYS_GID)_(MIN|MAX)' /etc/login.defs
```
Salida esperada:
```text
wheel:x:10:student
student:x:1000:
root:::
bin:::
daemon:::
UID_MIN                  1000
UID_MAX                 60000
SYS_UID_MIN               201
SYS_UID_MAX               999
GID_MIN                  1000
GID_MAX                 60000
SYS_GID_MIN               201
SYS_GID_MAX               999
```
Qué observar: `student:x:1000:` no lista miembros porque el grupo primario se registra en `passwd`, no en `group`. Los rangos confirman lo dicho en conceptos.

6. Quién está conectado y quién estuvo:

```bash
who
w
last -n 5
sudo lastb -n 5
lastlog | grep -v 'Never logged in'
```
Salida esperada (resumida):
```text
student  pts/0        2026-09-03 08:01 (10.0.2.2)

 08:20:11 up 25 min,  1 user,  load average: 0.00, 0.01, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
student  pts/0    10.0.2.2         08:01    1.00s  0.05s  0.01s w

student  pts/0        10.0.2.2         Wed Sep  3 08:01   still logged in
reboot   system boot  5.14.0-570.el9   Wed Sep  3 07:56   still running

btmp begins Wed Sep  3 07:56:02 2026

Username         Port     From             Latest
root             tty1                      Wed Sep  3 07:58:10 -0500 2026
student          pts/0    10.0.2.2         Wed Sep  3 08:01:22 -0500 2026
```
Qué observar: la IP `10.0.2.2` es la puerta de enlace NAT del hipervisor: así se ve una conexión que llega por el port forwarding. Es la misma dirección en VirtualBox (NAT) y en UTM con red compartida sobre QEMU; si algún participante configuró otro modo de red verá otra IP, y eso también es correcto. `lastlog` recorre **todas** las cuentas de `/etc/passwd` (las de servicio dicen `**Never logged in**`, por eso se filtran); la línea de `root` en `tty1` solo aparece si alguien entró como root en la consola del hipervisor. `lastb` estará casi vacío hoy; el Día 04 lo veremos lleno tras simular ataques de fuerza bruta.

- **Checkpoint:** pegar en el chat la salida de:
```bash
id; getent passwd student; sudo grep '^student' /etc/shadow | cut -d: -f1,3-8
```
Salida esperada (última línea): `student:20699:0:99999:7::`

## Bloque 2 — Ciclo de vida de usuarios y grupos

### Conceptos (15 min)

**Qué hace `useradd ana` por dentro.** Una línea en `passwd`, otra en `shadow` con `!!`, un grupo privado `ana` en `group` y `gshadow`, el directorio `/home/ana` copiado desde `/etc/skel` con modo `0700` (`HOME_MODE 0700` en `login.defs`; a diferencia de Debian/Ubuntu, en RHEL los homes nunca son legibles por otros usuarios) y un buzón en `/var/spool/mail/ana`. Los valores por defecto salen de dos archivos:
- `/etc/default/useradd` (se consulta con `useradd -D`): grupo por defecto, base de homes, shell, `skel`, inactividad.
- `/etc/login.defs`: rangos de UID/GID, `PASS_MAX_DAYS`, `PASS_WARN_AGE`, `CREATE_HOME`, `UMASK`, `HOME_MODE`, algoritmo de hash.

Cambiar estos archivos afecta solo a los usuarios que se creen **después**.

**Opciones de `useradd` que hay que dominar:**
- `-u UID`: fijar el UID (útil para que coincida entre servidores o con un NFS).
- `-g GRUPO`: grupo **primario** distinto del privado (no se crea grupo privado).
- `-G g1,g2`: grupos **suplementarios**.
- `-c "Comentario"`: nombre completo y área; evitar `:` y `,`.
- `-s /bin/bash` o `-s /sbin/nologin`: shell.
- `-m`: crear el home. En RHEL es el comportamiento por defecto (`CREATE_HOME yes`), pero se escribe igual: en otras distribuciones es obligatorio y en el examen no cuesta nada.
- `-d /ruta`: home distinto; `-e AAAA-MM-DD`: expiración de cuenta; `-r`: cuenta de sistema (UID < 1000, sin home, sin expiración).

**`passwd`.** Como root, `passwd ana` no pide la contraseña anterior y **acepta contraseñas débiles** (avisa "BAD PASSWORD" pero continúa). Un usuario normal cambiando la suya sí está sujeto a `pam_pwquality` (mínimo 8 caracteres por defecto en RHEL 9). `passwd --stdin` (extensión de Red Hat) lee la contraseña de la entrada estándar: sirve para scripts. `passwd -S` muestra el estado (`PS` = contraseña puesta, `LK` = bloqueada, `NP` = sin contraseña).

**`usermod`.** Modifica lo que `useradd` creó: `-c`, `-s`, `-u`, `-d -m` (mover el home), `-l` (renombrar), `-e` (expiración), `-L/-U` (bloquear/desbloquear contraseña) y el importante `-G`. **`usermod -G grupo` reemplaza la lista completa de grupos suplementarios**; `usermod -aG grupo` (append) añade. Olvidar el `-a` es el error más frecuente de este tema y en un servidor real deja a alguien sin acceso a todo lo demás. Alternativa sin riesgo: `gpasswd -a usuario grupo`. Los cambios de grupo se ven en la **siguiente sesión** del usuario: la sesión abierta conserva la lista de grupos con la que inició.

**`chage`.** Administra los campos 3 a 8 de `shadow`: `-l` (listar), `-M` (máximo de días), `-m` (mínimo), `-W` (aviso), `-I` (inactividad), `-E AAAA-MM-DD` (expiración de cuenta; `-E -1` la quita), `-d 0` (marca la contraseña como caducada: obliga a cambiarla en el próximo ingreso). `passwd -e` equivale a `chage -d 0`.

**Tres niveles de bloqueo, y no son intercambiables:**
1. `passwd -l` / `usermod -L`: bloquea **solo la contraseña**. Una llave SSH autorizada sigue entrando.
2. `usermod -s /sbin/nologin`: no hay shell interactiva; muestra "This account is currently not available". Otros servicios (correo, FTP, `su -c`) pueden seguir autenticando.
3. `chage -E 0` / `usermod -e AAAA-MM-DD`: la **cuenta** expira. PAM rechaza cualquier tipo de acceso. Es el bloqueo completo.

Práctica institucional recomendada al desvincular a alguien: expirar la cuenta y poner `nologin` el mismo día; borrar con `userdel -r` solo cuando lo permita la política de retención, y antes revisar qué archivos le pertenecen fuera de su home.

**`userdel -r`** borra la cuenta, su home y su buzón. Los archivos que tuviera en otros lugares quedan **huérfanos**: `ls -l` muestra el UID numérico. Si más adelante se crea otro usuario con ese UID, hereda esos archivos. Buscarlos con `find / -xdev -nouser`.

**Grupos.** `groupadd -g GID nombre`, `groupmod -n nuevo viejo`, `groupdel` (falla si es el grupo primario de alguien), `gpasswd -a usuario grupo` (añadir), `gpasswd -d usuario grupo` (quitar), `gpasswd -A usuario grupo` (nombrar administrador del grupo: puede añadir y quitar miembros sin ser root), `gpasswd -M lista grupo` (fijar la lista completa). `newgrp grupo` abre una shell con ese grupo como primario temporal (los archivos nuevos nacen con él).

### Lab 2.1 — Estructura institucional: grupos y usuarios (20 min)

- **Objetivo:** crear los grupos `sistemas`, `soporte` y `auditoria` y los usuarios `ana`, `carlos` (sistemas), `pedro` (soporte) y `laura` (auditoria) con UID/GID fijos, contraseña inicial y comprobación de acceso.

1. Ver los valores por defecto antes de crear nada:

```bash
sudo useradd -D
grep -Ev '^#|^$' /etc/login.defs | grep -E 'PASS_|CREATE_HOME|UMASK|HOME_MODE|USERGROUPS|ENCRYPT'
ls -la /etc/skel
```
Salida esperada:
```text
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
PASS_MAX_DAYS	99999
PASS_MIN_DAYS	0
PASS_MIN_LEN	5
PASS_WARN_AGE	7
CREATE_HOME	yes
UMASK           022
HOME_MODE       0700
USERGROUPS_ENAB yes
ENCRYPT_METHOD SHA512
total 24
drwxr-xr-x.  2 root root   62 Sep  1 10:02 .
drwxr-xr-x. 91 root root 8192 Sep  3 09:10 ..
-rw-r--r--.  1 root root   18 Feb 15  2024 .bash_logout
-rw-r--r--.  1 root root  141 Feb 15  2024 .bash_profile
-rw-r--r--.  1 root root  492 Feb 15  2024 .bashrc
```
Qué observar: `useradd -D` puede mostrar también `LOG_INIT=yes` según la versión de `shadow-utils`. Lo que haya en `/etc/skel` se copia a cada home nuevo: es el lugar para dejar un `.bashrc` institucional.

2. Crear los grupos con GID fijo (siempre **antes** que los usuarios que los usan):

```bash
sudo groupadd -g 3001 sistemas
sudo groupadd -g 3002 soporte
sudo groupadd -g 3003 auditoria
getent group sistemas soporte auditoria
```
Salida esperada:
```text
sistemas:x:3001:
soporte:x:3002:
auditoria:x:3003:
```

3. Crear los usuarios con UID fijo, comentario y grupo suplementario:

```bash
sudo useradd -m -u 2001 -c "Ana Rodriguez - Sistemas" -G sistemas ana
sudo useradd -m -u 2002 -c "Carlos Mendez - Sistemas" -G sistemas carlos
sudo useradd -m -u 2003 -c "Pedro Castillo - Soporte" -G soporte pedro
sudo useradd -m -u 2004 -c "Laura Gomez - Auditoria" -G auditoria laura
for u in ana carlos pedro laura; do id $u; done
getent group sistemas soporte auditoria
ls -ld /home/*
```
Salida esperada:
```text
uid=2001(ana) gid=2001(ana) groups=2001(ana),3001(sistemas)
uid=2002(carlos) gid=2002(carlos) groups=2002(carlos),3001(sistemas)
uid=2003(pedro) gid=2003(pedro) groups=2003(pedro),3002(soporte)
uid=2004(laura) gid=2004(laura) groups=2004(laura),3003(auditoria)
sistemas:x:3001:ana,carlos
soporte:x:3002:pedro
auditoria:x:3003:laura
drwx------. 2 ana     ana      62 Sep  3 09:32 /home/ana
drwx------. 2 carlos  carlos   62 Sep  3 09:32 /home/carlos
drwx------. 2 laura   laura    62 Sep  3 09:32 /home/laura
drwx------. 2 pedro   pedro    62 Sep  3 09:32 /home/pedro
drwx------. 5 student student 95 Sep  3 08:01 /home/student
```
Qué observar: cada usuario tiene su grupo privado con GID igual al UID; los homes nacen `700` (el contador de enlaces y el tamaño de `/home/student` varían según lo creado los Días 01 y 02). Nadie más que el dueño (y root) entra a un home en RHEL 9.

4. Todavía no pueden entrar: sin contraseña no hay acceso.

```bash
sudo grep '^ana' /etc/shadow
sudo passwd -S ana
```
Salida esperada:
```text
ana:!!:20699:0:99999:7:::
ana LK 2026-09-03 0 99999 7 -1 (Password locked.)
```

5. Asignar contraseñas. La primera de forma interactiva (así se ve el aviso de calidad), el resto con `--stdin`. Para que las demostraciones del día sean predecibles, **todos los usuarios de práctica usan `Pgn.2026`**:

```bash
sudo passwd ana
```
Salida esperada (escribir `Pgn.2026` dos veces):
```text
Changing password for user ana.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```
```bash
for u in carlos pedro laura; do echo 'Pgn.2026' | sudo passwd --stdin $u; done
for u in ana carlos pedro laura; do sudo passwd -S $u; done
```
Salida esperada:
```text
Changing password for user carlos.
passwd: all authentication tokens updated successfully.
Changing password for user pedro.
passwd: all authentication tokens updated successfully.
Changing password for user laura.
passwd: all authentication tokens updated successfully.
ana PS 2026-09-03 0 99999 7 -1 (Password set, SHA512 crypt.)
carlos PS 2026-09-03 0 99999 7 -1 (Password set, SHA512 crypt.)
pedro PS 2026-09-03 0 99999 7 -1 (Password set, SHA512 crypt.)
laura PS 2026-09-03 0 99999 7 -1 (Password set, SHA512 crypt.)
```
Qué observar: si se prueba una contraseña corta como root, aparece `BAD PASSWORD: The password is shorter than 8 characters` pero se acepta igual. Decirlo en clase: root puede saltarse la política; el usuario no. El `for` no es capricho: el `passwd` de RHEL acepta **una sola** cuenta por invocación (`passwd -S ana carlos` responde `Only one user name may be specified.`).

6. Comprobar que ana puede iniciar sesión (se pide **su** contraseña, no la de student):

```bash
su - ana
```
Dentro de la sesión de ana:
```bash
id
pwd
touch mi-archivo.txt
ls -l
exit
```
Salida esperada:
```text
Password:
[ana@rhel01 ~]$ id
uid=2001(ana) gid=2001(ana) groups=2001(ana),3001(sistemas) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[ana@rhel01 ~]$ pwd
/home/ana
[ana@rhel01 ~]$ ls -l
total 0
-rw-rw-r--. 1 ana ana 0 Sep  3 09:40 mi-archivo.txt
```
Qué observar: el prompt cambia a `[ana@rhel01 ~]$`. El archivo nace con grupo `ana` (su primario) y modo `664`; ese modo lo explica el `umask` en el Bloque 4. `exit` devuelve a student.

7. **El error clásico, a propósito.** Supongamos que alguien quiere que carlos también apoye a soporte y escribe `-G` sin `-a`:

```bash
sudo usermod -G soporte carlos
id carlos
```
Salida esperada:
```text
uid=2002(carlos) gid=2002(carlos) groups=2002(carlos),3002(soporte)
```
Qué observar: **carlos perdió `sistemas`.** En un servidor real, esto es un ticket "ya no puedo entrar a la carpeta del área". Reparar y dejar a carlos como estaba:

```bash
sudo usermod -aG sistemas carlos
id carlos
sudo gpasswd -d carlos soporte
id carlos
```
Salida esperada:
```text
uid=2002(carlos) gid=2002(carlos) groups=2002(carlos),3001(sistemas),3002(soporte)
Removing user carlos from group soporte
uid=2002(carlos) gid=2002(carlos) groups=2002(carlos),3001(sistemas)
```

8. *(Opcional — "Si sobra tiempo"; si el grupo va atrasado, el instructor lo demuestra y se pasa al paso 9.)* Delegar la administración del grupo `sistemas` a ana (sin darle root):

```bash
sudo gpasswd -A ana sistemas
sudo grep '^sistemas' /etc/gshadow
sudo -u ana gpasswd -a laura sistemas
getent group sistemas
sudo -u ana gpasswd -d laura sistemas
getent group sistemas
```
Salida esperada:
```text
sistemas:!:ana:ana,carlos
Adding user laura to group sistemas
sistemas:x:3001:ana,carlos,laura
Removing user laura from group sistemas
sistemas:x:3001:ana,carlos
```
Qué observar: el tercer campo de `gshadow` es la lista de administradores. `sudo -u ana` ejecuta el comando como ana (student puede hacerlo porque su regla `wheel` dice `(ALL)`).

9. Grupo primario distinto con `-g` (sin grupo privado). Creamos una cuenta temporal que borraremos en el Lab 2.2:

```bash
sudo useradd -m -u 2099 -g soporte -c "Practicante temporal" temporal
echo 'Pgn.2026' | sudo passwd --stdin temporal
id temporal
getent group temporal || echo "no existe grupo privado 'temporal'"
```
Salida esperada:
```text
Changing password for user temporal.
passwd: all authentication tokens updated successfully.
uid=2099(temporal) gid=3002(soporte) groups=3002(soporte)
no existe grupo privado 'temporal'
```

10. *(Opcional — "Si sobra tiempo"; puede quedar como demo del instructor.)* `newgrp`: cambiar el grupo primario durante una sesión.

```bash
sudo su - carlos
```
Como carlos:
```bash
id -gn
touch /tmp/a-carlos.txt
newgrp sistemas
id -gn
touch /tmp/b-carlos.txt
ls -l /tmp/*-carlos.txt
exit
exit
```
Salida esperada:
```text
carlos
sistemas
-rw-rw-r--. 1 carlos carlos   0 Sep  3 09:48 /tmp/a-carlos.txt
-rw-r--r--. 1 carlos sistemas 0 Sep  3 09:48 /tmp/b-carlos.txt
```
Qué observar: `newgrp` abre una shell nueva (por eso hay dos `exit`). No pide contraseña porque carlos es miembro. El segundo archivo nace con grupo `sistemas`, pero fíjese en el modo: `644` en vez de `664`. La shell nueva vuelve a leer `/etc/bashrc`, que asigna `umask 002` **solo** cuando el grupo primario tiene el mismo nombre que el usuario (grupo privado); como ahora el primario es `sistemas`, aplica `022`. Es un adelanto del `umask` del Bloque 4. `newgrp` sirve para que un archivo nazca con el grupo del área sin tocar el directorio; en el Bloque 4 veremos la forma automática (setgid).

- **Checkpoint:** pegar la salida de:
```bash
for u in ana carlos pedro laura temporal; do id $u; done; getent group sistemas soporte auditoria
```
Salida esperada (última parte): `sistemas:x:3001:ana,carlos`, `soporte:x:3002:pedro`, `auditoria:x:3003:laura`; `temporal` con `gid=3002(soporte)`. Si alguien restauró un snapshot o se atrasó, el instructor le pasa `crear-usuarios.sh` (Lab 2.3) para ponerse al día.

### Lab 2.2 — Política de contraseñas y bloqueo de cuentas (15 min)

- **Objetivo:** aplicar la política institucional (contraseña expira a los 90 días, aviso 7 días antes, mínimo 1 día entre cambios), forzar el cambio en el primer ingreso y practicar los tres niveles de bloqueo y el borrado.

1. Estado inicial de ana:

```bash
sudo chage -l ana
```
Salida esperada:
```text
Last password change					: Sep 03, 2026
Password expires					: never
Password inactive					: never
Account expires						: never
Minimum number of days between password change		: 0
Maximum number of days between password change		: 99999
Number of days of warning before password expires	: 7
```

2. Aplicar la política a los cuatro usuarios de área:

```bash
for u in ana carlos pedro laura; do sudo chage -M 90 -m 1 -W 7 $u; done
sudo chage -l ana
sudo grep '^ana' /etc/shadow | cut -d: -f1,3-6
```
Salida esperada:
```text
Last password change					: Sep 03, 2026
Password expires					: Dec 02, 2026
Password inactive					: never
Account expires						: never
Minimum number of days between password change		: 1
Maximum number of days between password change		: 90
Number of days of warning before password expires	: 7
ana:20699:1:90:7
```
Qué observar: `chage -l` es la vista humana; `shadow` es la vista cruda. `-m 1` impide que un usuario cambie la contraseña diez veces seguidas para volver a la anterior.

3. Que los usuarios **futuros** nazcan con la política (solo afecta a los que se creen después):

```bash
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS\t90/' /etc/login.defs
grep '^PASS_' /etc/login.defs
```
Salida esperada:
```text
PASS_MAX_DAYS	90
PASS_MIN_DAYS	0
PASS_MIN_LEN	5
PASS_WARN_AGE	7
```

4. Forzar cambio de contraseña en el próximo ingreso (alta de personal: el administrador entrega una clave inicial y el usuario debe cambiarla):

```bash
sudo chage -d 0 laura
sudo chage -l laura | head -1
su - laura
```
Salida esperada:
```text
Last password change					: password must be changed
Password:
You are required to change your password immediately (administrator enforced).
Current password:
```
Qué observar: tras escribir `Pgn.2026`, PAM exige una contraseña nueva antes de entregar la shell. Para mantener la contraseña del curso, cancelar con `Ctrl+C` y restaurar la fecha de cambio a hoy:

```bash
sudo chage -d "$(date +%F)" laura
sudo chage -l laura | head -1
```
Salida esperada:
```text
Last password change					: Sep 03, 2026
```
Si algún participante llegó a completar el cambio y ya no sabe la clave de laura, se restaura la del curso con `echo 'Pgn.2026' | sudo passwd --stdin laura` (y de nuevo `sudo chage -d "$(date +%F)" laura`).

5. **Nivel 1: bloquear la contraseña** de pedro y comprobar que no entra:

```bash
sudo passwd -l pedro
sudo passwd -S pedro
sudo grep '^pedro' /etc/shadow | cut -d: -f2 | cut -c1-5
su - pedro
```
Salida esperada:
```text
Locking password for user pedro.
passwd: Success
pedro LK 2026-09-03 1 90 7 -1 (Password locked.)
!!$6$
Password:
su: Authentication failure
```
Qué observar: el hash sigue en `shadow`; solo se le antepuso `!!` (`usermod -L pedro` hace lo mismo pero con un solo `!`; ambos invalidan el hash, que ya no empieza por `$`). Desbloquear:

```bash
sudo passwd -u pedro
sudo passwd -S pedro
```
Salida esperada:
```text
Unlocking password for user pedro.
passwd: Success
pedro PS 2026-09-03 1 90 7 -1 (Password set, SHA512 crypt.)
```

6. **Nivel 2: shell sin acceso** para la cuenta temporal:

```bash
sudo usermod -s /sbin/nologin temporal
getent passwd temporal
su - temporal
```
Salida esperada:
```text
temporal:x:2099:3002:Practicante temporal:/home/temporal:/sbin/nologin
Password:
This account is currently not available.
```
Qué observar: la contraseña fue correcta (no dice "Authentication failure"); la shell `nologin` simplemente imprime el mensaje y termina. `/sbin/nologin` es la shell de las cuentas de servicio y **a propósito no aparece** en `/etc/shells` (compruébelo con `cat /etc/shells`): los servicios que exigen una shell válida —FTP, por ejemplo— consultan esa lista, así que ese usuario también queda fuera de ellos. Ojo: correo y `sudo -u` no la consultan, por eso este bloqueo es de "nivel 2" y no total.

7. **Nivel 3: expirar la cuenta** completa:

```bash
sudo chage -E 0 temporal
sudo chage -l temporal | grep 'Account expires'
su - temporal
```
Salida esperada:
```text
Account expires						: Jan 01, 1970
Password:
Your account has expired; please contact your system administrator.
```
Qué observar: PAM rechaza el acceso antes de mirar la shell. Ni contraseña, ni llave SSH, ni ningún servicio: la cuenta está cerrada. `chage -E -1 temporal` la reabriría.

8. Borrar la cuenta y ver qué pasa con sus archivos fuera del home. Primero se le fabrica un archivo fuera de `/home/temporal` (se crea como root y se le asigna a `temporal`, porque la cuenta ya está expirada y no debe ejecutar nada):

```bash
sudo install -o temporal -g soporte -m 644 /dev/null /tmp/huerfano.txt
ls -l /tmp/huerfano.txt
sudo userdel -r temporal
getent passwd temporal || echo "temporal eliminado"
ls -ld /home/temporal
ls -l /tmp/huerfano.txt
sudo find / -xdev -nouser 2>/dev/null
sudo rm /tmp/huerfano.txt
```
Salida esperada:
```text
-rw-r--r--. 1 temporal soporte 0 Sep  3 10:02 /tmp/huerfano.txt
temporal eliminado
ls: cannot access '/home/temporal': No such file or directory
-rw-r--r--. 1 2099 soporte 0 Sep  3 10:02 /tmp/huerfano.txt
/tmp/huerfano.txt
```
Qué observar: `ls` ya no puede traducir el UID 2099 y muestra el número (si `find` lista alguna otra ruta, es otro huérfano previo del sistema; hoy solo importa el nuestro). Si mañana se crea un usuario con `-u 2099`, ese archivo será suyo. Por eso antes de borrar se ejecuta `find / -xdev -user temporal` y se reasigna o archiva.

- **Checkpoint:** pegar la salida de:
```bash
sudo chage -l ana | head -2; sudo passwd -S pedro; getent passwd temporal || echo "temporal eliminado"
```
Salida esperada: `Password expires : Dec 02, 2026`, `pedro PS ... 1 90 7 -1 (Password set, SHA512 crypt.)` y `temporal eliminado`.

### Lab 2.3 — Opcional: automatizar con `crear-usuarios.sh` (10 min, fuera de agenda: solo si sobra tiempo o como demo)

- **Objetivo:** dejar un script idempotente que reconstruye grupos, usuarios y política. Sirve para restaurar el estado del curso tras volver a un snapshot y es el punto de partida del Día 07 (Bash scripting). Hoy solo se lee y se ejecuta; no se explica la sintaxis.

1. Crear el archivo `~/crear-usuarios.sh` con `nano` o `vi` y pegar:

```bash
#!/bin/bash
# crear-usuarios.sh - Crea grupos y usuarios institucionales del curso (Dia 03).
# Uso: sudo ./crear-usuarios.sh
# Es idempotente: si el grupo o el usuario ya existe, lo deja como esta.

PASS_INICIAL='Pgn.2026'

# grupo:gid
GRUPOS="sistemas:3001 soporte:3002 auditoria:3003"

# usuario:uid:grupo_suplementario:comentario
USUARIOS="
ana:2001:sistemas:Ana Rodriguez - Sistemas
carlos:2002:sistemas:Carlos Mendez - Sistemas
pedro:2003:soporte:Pedro Castillo - Soporte
laura:2004:auditoria:Laura Gomez - Auditoria
"

if [ "$(id -u)" -ne 0 ]; then
    echo "Ejecutar con sudo" >&2
    exit 1
fi

for entrada in $GRUPOS; do
    grupo=${entrada%%:*}
    gid=${entrada##*:}
    if getent group "$grupo" > /dev/null; then
        echo "Grupo $grupo ya existe"
    else
        groupadd -g "$gid" "$grupo"
        echo "Grupo $grupo creado (GID $gid)"
    fi
done

while IFS=: read -r usuario uid grupo comentario; do
    [ -z "$usuario" ] && continue
    if id "$usuario" &> /dev/null; then
        echo "Usuario $usuario ya existe"
        continue
    fi
    useradd -m -u "$uid" -G "$grupo" -c "$comentario" -s /bin/bash "$usuario"
    echo "$PASS_INICIAL" | passwd --stdin "$usuario" > /dev/null
    chage -M 90 -m 1 -W 7 "$usuario"
    echo "Usuario $usuario creado (UID $uid, grupo $grupo)"
done <<< "$USUARIOS"
```

2. Dar permiso de ejecución y ejecutarlo (con los usuarios ya creados, debe informar que existen):

```bash
chmod u+x ~/crear-usuarios.sh
sudo ~/crear-usuarios.sh
```
Salida esperada:
```text
Grupo sistemas ya existe
Grupo soporte ya existe
Grupo auditoria ya existe
Usuario ana ya existe
Usuario carlos ya existe
Usuario pedro ya existe
Usuario laura ya existe
```
Qué observar: el script no rompe nada si se ejecuta dos veces. Para probar la rama "creado", el instructor puede borrar a laura (`sudo userdel -r laura`) y volver a ejecutarlo.

- **Checkpoint:** pegar la salida de:
```bash
sudo ~/crear-usuarios.sh | tail -4; id laura
```
Salida esperada: cuatro líneas `Usuario ... ya existe` (o `creado`, si se borró alguna cuenta) y `uid=2004(laura) gid=2004(laura) groups=2004(laura),3003(auditoria)`.

## Bloque 3 — Privilegios: `su`, `sudo` y `sudoers`

### Conceptos (10 min)

**Tres caminos hacia root.**
1. Entrar directamente como root: evitar. En RHEL 9 `sshd` trae `PermitRootLogin prohibit-password` (solo con llave), salvo que en el instalador se haya marcado "Allow root SSH login with password" (crea `/etc/ssh/sshd_config.d/01-permitrootlogin.conf`). En consola local debería usarse solo en emergencias.
2. `su -`: pide la **contraseña de root**. Es un secreto compartido: el log dice "alguien hizo su", no quién ni qué ejecutó después.
3. `sudo`: pide la **contraseña propia**, registra cada comando con el usuario que lo pidió, permite delegar comandos concretos y se revoca quitando una línea. Es el estándar institucional y lo que evalúa el RHCSA.

**`su` vs `su -`.** Sin el guion, `su ana` cambia de identidad pero conserva el entorno del que llama: directorio actual, variables, `PATH`. Con el guion (`su -`, `su -l`, `su - ana`) se abre una **shell de login**: se leen `/etc/profile` y los archivos de ana, se va a su home, el entorno es el que ella tendría al conectarse. Regla práctica: usar siempre el guion. `su -c 'comando' ana` ejecuta un solo comando. Cuando root hace `su - ana` no se pide contraseña: por eso hoy usamos `sudo su - usuario` para demostrar permisos sin escribir claves cada vez.

**`sudo` y `/etc/sudoers`.** Se edita **solo con `visudo`**: bloquea el archivo, valida la sintaxis al guardar y, si hay un error, pregunta antes de dejar un `sudoers` roto (un `sudoers` roto deja a **todos** sin `sudo`; la salida es `su -` con la contraseña de root). Las reglas propias van en `/etc/sudoers.d/` (un archivo por delegación, sin punto ni `~` en el nombre, creado con `visudo -f`): las actualizaciones de paquetes no los tocan y se revocan borrando el archivo. La línea `#includedir /etc/sudoers.d` (o `@includedir` en versiones recientes) al final de `sudoers` los carga; ese `#` **no** es un comentario.

Anatomía de una regla:

```text
%soporte   ALL=(root)   /usr/bin/systemctl restart httpd
 quién     dónde  como quién       qué (ruta absoluta y argumentos exactos)
```
- `quién`: usuario, `%grupo` o `User_Alias`.
- `dónde`: nombre de host; `ALL` = cualquiera (importa si el archivo se replica a varios servidores).
- `(como quién)`: `(root)`, `(ALL)`, `(ALL:ALL)` para también cambiar de grupo.
- `qué`: lista de comandos o `Cmnd_Alias`. Si se escriben argumentos, `sudo` exige que coincidan exactamente; sin argumentos, se permite cualquiera.
- `NOPASSWD:` delante del comando: sin contraseña. Criterio: solo para automatización (cron, monitoreo) o comandos de consulta sin efectos. **Nunca `NOPASSWD: ALL`** en una cuenta interactiva.
- Peligro clásico: delegar comandos que abren una shell o un editor (`vim`, `less`, `find -exec`, `bash`, `python`). Con `sudo vim` se obtiene root con `:!bash`. Un pentester lo busca primero (`sudo -l`).

**Grupo `wheel` en RHEL.** `/etc/sudoers` trae `%wheel ALL=(ALL) ALL`: todo miembro de `wheel` puede ejecutar todo como cualquiera, con su contraseña. El instalador metió a `student` en `wheel`. Dar sudo completo a alguien es simplemente `usermod -aG wheel usuario`.

**Uso diario:** `sudo -l` (qué puedo hacer), `sudo -i` (shell de login de root con mi contraseña), `sudo -u ana comando`, `sudo -k` (olvidar la credencial cacheada; por defecto dura 5 minutos: `timestamp_timeout`). **Auditoría:** cada uso queda en `/var/log/secure` (vía rsyslog, facility `authpriv`) y en el journal (`journalctl -t sudo`). Los rechazos aparecen como `command not allowed` o `user NOT in sudoers`.

### Lab 3.1 — Delegación limitada para el grupo soporte (20 min)

- **Objetivo:** ver la diferencia de entorno entre `su` y `su -`, comprobar el estado de `sudo` para usuarios sin privilegios, crear una regla que permita al grupo `soporte` reiniciar y consultar servicios concretos, y auditar todo en los logs.

1. `su` vs `su -` (se pide la contraseña de ana en ambos):

```bash
su ana -c 'pwd; echo "HOME=$HOME MAIL=$MAIL"'
su - ana -c 'pwd; echo "HOME=$HOME MAIL=$MAIL"'
```
Salida esperada:
```text
Password:
/home/student
HOME=/home/ana MAIL=/var/spool/mail/student
Password:
/home/ana
HOME=/home/ana MAIL=/var/spool/mail/ana
```
Qué observar: sin guion, `su` solo reemplaza `HOME`, `SHELL`, `USER` y `LOGNAME`; el resto del entorno (directorio actual, `MAIL`, variables propias) sigue siendo el de student: ana "hereda" el buzón de student. (`MAIL` lo fija `/etc/profile` con `MAIL="/var/spool/mail/$USER"`, y `/etc/profile` solo se lee en shells de login.) Con guion se leen `/etc/profile` y los archivos de ana: el entorno es el de un login real.

2. Shell de root con `sudo -i` (contraseña de student) frente a `su -` (contraseña de root):

```bash
sudo -i
```
Como root:
```bash
whoami; pwd; umask; exit
```
Salida esperada:
```text
root
/root
0022
```

3. Dónde está la regla de `wheel` y quién puede leer `sudoers`:

```bash
sudo grep -nE 'wheel|includedir' /etc/sudoers
ls -l /etc/sudoers; ls -ld /etc/sudoers.d
```
Salida esperada:
```text
105:## Allows people in group wheel to run all commands
106:%wheel	ALL=(ALL)	ALL
109:# %wheel	ALL=(ALL)	NOPASSWD: ALL
118:## Read drop-in files from /etc/sudoers.d (the # here does not mean a comment)
119:#includedir /etc/sudoers.d
-r--r-----. 1 root root 4328 Sep  1 10:02 /etc/sudoers
drwxr-x---. 2 root root   26 Sep  1 10:02 /etc/sudoers.d
```
Qué observar: los números de línea pueden variar. `sudoers` es `440`: ni siquiera root lo escribe sin `visudo` (que lo hace por él).

4. Estado actual de pedro y laura (sin privilegios):

```bash
su - pedro
```
Como pedro:
```bash
sudo -l
sudo cat /etc/shadow
exit
```
Salida esperada:
```text
[sudo] password for pedro:
Sorry, user pedro may not run sudo on rhel01.
pedro is not in the sudoers file.  This incident will be reported.
```
Qué observar: `sudo` siempre pide la contraseña **antes** de decidir; así no revela a un atacante si la cuenta tiene privilegios. La segunda vez puede no pedirla: la credencial queda en caché 5 minutos aunque la respuesta sea negativa. "This incident will be reported" significa: quedó en `/var/log/secure`.

5. Crear la regla en `/etc/sudoers.d/soporte` con `visudo -f` (abre `vi`):

```bash
sudo visudo -f /etc/sudoers.d/soporte
```
Quien prefiera `nano` **no** puede escribir `sudo EDITOR=nano visudo ...`: con la configuración por defecto (`Defaults env_reset` y sin la etiqueta `SETENV`) sudo responde `sudo: sorry, you are not allowed to set the following environment variables: EDITOR`. La forma que sí funciona es abrir primero una shell de root y exportar la variable dentro de ella:

```bash
sudo -i
EDITOR=nano visudo -f /etc/sudoers.d/soporte
exit
```
⚠️ Verificar en la VM antes de la clase: que este `visudo` respeta `$EDITOR` (opción `env_editor`, activada en la compilación de Red Hat). Se comprueba dentro de `sudo -i` con `sudo -V | grep -i editor`: debe aparecer la línea `Visudo will honor the EDITOR environment variable`. Si no aparece, `visudo` abrirá `/usr/bin/vi` sin avisar.
Contenido completo del archivo:
```text
## /etc/sudoers.d/soporte
## Delegacion limitada para el grupo soporte (mesa de ayuda).
## Editar SOLO con: sudo visudo -f /etc/sudoers.d/soporte

Cmnd_Alias SOPORTE_SERVICIOS = /usr/bin/systemctl restart chronyd, \
                               /usr/bin/systemctl status chronyd, \
                               /usr/bin/systemctl restart httpd, \
                               /usr/bin/systemctl status httpd

%soporte ALL=(root) SOPORTE_SERVICIOS

## Ejemplo de NOPASSWD con criterio (solo consulta, sin efectos). Descomentar si se desea:
## %soporte ALL=(root) NOPASSWD: /usr/bin/systemctl status chronyd
```
Guardar (`Esc`, `:wq`). Validar y revisar permisos:

```bash
sudo visudo -c
ls -l /etc/sudoers.d/soporte
```
Salida esperada:
```text
/etc/sudoers: parsed OK
/etc/sudoers.d/soporte: parsed OK
-r--r-----. 1 root root 512 Sep  3 10:25 /etc/sudoers.d/soporte
```
Qué observar: `httpd` aún no está instalado (se instala el Día 04); la regla es válida igual. Si `visudo` detecta un error muestra `>>> /etc/sudoers.d/soporte: syntax error near line N <<<` y pregunta `What now?`: `e` vuelve al editor, `x` sale **sin guardar**, `Q` guarda el archivo roto (no usar).

6. Probar como pedro:

```bash
su - pedro
```
Como pedro:
```bash
sudo -l
sudo systemctl restart chronyd
systemctl is-active chronyd
sudo systemctl restart sshd
sudo systemctl restart httpd
sudo cat /etc/shadow
exit
```
Salida esperada (resumida):
```text
[sudo] password for pedro:
Matching Defaults entries for pedro on rhel01:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, ...

User pedro may run the following commands on rhel01:
    (root) /usr/bin/systemctl restart chronyd, /usr/bin/systemctl status chronyd, /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd
active
Sorry, user pedro is not allowed to execute '/usr/bin/systemctl restart sshd' as root on rhel01.
Failed to restart httpd.service: Unit httpd.service not found.
Sorry, user pedro is not allowed to execute '/usr/bin/cat /etc/shadow' as root on rhel01.
```
Qué observar: el reinicio de `chronyd` no imprime nada (éxito silencioso); `is-active` no necesita sudo. Con `httpd`, **sudo sí lo permitió** y fue systemd quien falló: distinguir "no tengo permiso" de "el comando falló". Nótese que `sudo systemctl status chronyd --no-pager` sería rechazado: los argumentos deben coincidir exactamente.

7. Laura sigue sin nada:

```bash
sudo -l -U laura
sudo -l -U pedro | tail -2
```
Salida esperada:
```text
User laura is not allowed to run sudo on rhel01.
User pedro may run the following commands on rhel01:
    (root) /usr/bin/systemctl restart chronyd, /usr/bin/systemctl status chronyd, /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd
```
Qué observar: `sudo -l -U usuario` permite a un administrador auditar privilegios sin conocer la contraseña del otro.

8. El rastro en los logs:

```bash
sudo grep 'sudo\[' /var/log/secure | tail -n 6
sudo journalctl -t sudo --since "1 hour ago" --no-pager | tail -n 4
sudo grep -E 'FAILED SU|su-l' /var/log/secure | tail -n 3
```
Salida esperada (resumida):
```text
Sep  3 10:21:40 rhel01 sudo[3210]:    pedro : user NOT in sudoers ; TTY=pts/0 ; PWD=/home/pedro ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
Sep  3 10:31:02 rhel01 sudo[3390]:    pedro : TTY=pts/0 ; PWD=/home/pedro ; USER=root ; COMMAND=/usr/bin/systemctl restart chronyd
Sep  3 10:31:20 rhel01 sudo[3402]:    pedro : command not allowed ; TTY=pts/0 ; PWD=/home/pedro ; USER=root ; COMMAND=/usr/bin/systemctl restart sshd
Sep  3 10:31:35 rhel01 sudo[3415]:    pedro : TTY=pts/0 ; PWD=/home/pedro ; USER=root ; COMMAND=/usr/bin/systemctl restart httpd
Sep  3 10:31:50 rhel01 sudo[3427]:    pedro : command not allowed ; TTY=pts/0 ; PWD=/home/pedro ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
Sep  3 10:05:12 rhel01 su[2980]: FAILED SU (to pedro) student on pts/0
Sep  3 10:05:12 rhel01 su[2980]: pam_unix(su-l:auth): authentication failure; logname=student uid=1000 euid=0 tty=pts/0 ruser=student rhost=  user=pedro
```
Qué observar: cada línea dice **quién**, **desde dónde**, **como quién** y **qué**. El `FAILED SU` es el intento contra pedro bloqueado del Lab 2.2. Esto es lo que se entrega en una auditoría.

- **Checkpoint:** pegar la salida de:
```bash
sudo visudo -c; sudo -l -U pedro | tail -1
```
Salida esperada: dos líneas `parsed OK` y `(root) /usr/bin/systemctl restart chronyd, /usr/bin/systemctl status chronyd, /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd`.

## Bloque 4 — Permisos estándar, especiales y ACL

### Conceptos (20 min)

**Anatomía de `ls -l`:**

```text
-rwxr-x---. 1 ana sistemas 25 Sep  3 11:00 plan.txt
│└┬┘└┬┘└┬┘│ │  │    │      │
│ u   g   o │ │  dueño grupo  tamaño
│           │ enlaces
│           . = contexto SELinux    + = tiene ACL
└ tipo: - archivo, d directorio, l enlace, c/b dispositivo, s socket, p pipe
```
El kernel evalúa **una sola** terna: si el proceso es el dueño, aplica `u` y no mira más; si no, y pertenece al grupo, aplica `g`; si no, `o`. Consecuencia: un archivo `----rwx---` (dueño sin permisos, grupo con todo) es **ilegible para su propio dueño** aunque esté en el grupo. root ignora rwx, con una excepción: no puede ejecutar un archivo que no tenga ningún bit `x`.

**Octal.** `r=4 w=2 x=1`, se suma por terna. Los que hay que reconocer de un vistazo:

| Octal | Simbólico | Uso típico |
|---|---|---|
| 755 | `rwxr-xr-x` | directorios públicos, programas |
| 644 | `rw-r--r--` | archivos de configuración, documentos legibles |
| 700 | `rwx------` | home en RHEL 9, `.ssh` |
| 600 | `rw-------` | llaves privadas, archivos secretos |
| 750 | `rwxr-x---` | directorio de un área, lectura para el grupo |
| 770 | `rwxrwx---` | directorio colaborativo (sin herencia de grupo) |
| 2770 | `rwxrws---` | directorio colaborativo **con** setgid |
| 1777 | `rwxrwxrwt` | `/tmp`, buzón público con sticky |
| 4755 | `rwsr-xr-x` | binario setuid (`/usr/bin/passwd`) |

**`chmod` simbólico:** `quién` (`u g o a`) + `operación` (`+ - =`) + `permiso` (`r w x X s t`). Ejemplos: `u+x`, `g-w`, `o=r`, `a+r`, `u=rw,g=r,o=`, `-R` recursivo. La `X` mayúscula pone `x` solo a directorios y a archivos que ya eran ejecutables: `chmod -R g+rX` es la forma segura de dar lectura recursiva sin volver ejecutables los documentos.

**Permisos en directorios (esto es lo que más confunde):** `r` = listar nombres; `x` = entrar y acceder a lo que hay dentro **si se conoce el nombre**; `w` (junto con `x`) = crear, renombrar y **borrar** entradas. Borrar un archivo no depende de los permisos del archivo, sino de los del directorio que lo contiene. Un directorio `700` protege todo lo que hay dentro sin importar los permisos de cada archivo: por eso en RHEL 9 nadie ve el home de otro.

**`chown` y `chgrp`.** `chown ana archivo`, `chown ana:sistemas archivo`, `chown :sistemas archivo`, `chgrp sistemas archivo`, `-R` recursivo. Solo root puede cambiar el dueño (regalar un archivo permitiría burlar cuotas y setuid). El dueño puede cambiar el grupo únicamente a uno al que pertenece.

**`umask`.** Máscara que se **resta** al modo base cuando se crea algo: `666` para archivos, `777` para directorios (bash nunca crea archivos ejecutables). Con `022`: archivos `644`, directorios `755`. Con `002`: `664`/`775`. Con `077`: `600`/`700`. **No asumir el valor: comprobarlo con `umask`.** En RHEL 9 lo fijan `/etc/profile` (shells de login) y `/etc/bashrc` (el resto) con la **misma** regla: root y cuentas de sistema (UID ≤ 199) obtienen `0022`; un usuario normal cuyo grupo primario tiene su mismo nombre (el grupo privado) obtiene `0002`, porque nadie más está en ese grupo y el `w` de grupo es inofensivo hasta que se usa setgid. Para hacerlo persistente por usuario: `~/.bashrc`; para todo el sistema: un archivo en `/etc/profile.d/`.

**Permisos especiales (el cuarto dígito octal):**
- **setuid (4)**: el programa corre con el UID del **dueño** del archivo, no del que lo ejecuta. `/usr/bin/passwd` es de root y setuid: así un usuario escribe su hash en `/etc/shadow` (modo `000`). Se ve como `s` en la `x` del dueño (`rws`). Linux lo **ignora** en scripts y en directorios. Inventario: `find / -perm -4000 -type f`. Un setuid nuevo e inesperado es una de las primeras cosas que revisa un análisis forense.
- **setgid (2)**: en un ejecutable, corre con el GID del grupo dueño. En un **directorio**, todo lo que se crea dentro hereda el grupo del directorio (no el primario del creador) y los subdirectorios nacen con setgid también. Es la base de las carpetas de área. Se ve como `s` en la `x` del grupo (`rws`).
- **sticky (1)**: en un directorio, solo el dueño del archivo (o del directorio, o root) puede borrar o renombrar cada entrada, aunque el directorio sea `777`. `/tmp` es `1777`. Se ve como `t` en la `x` de otros (`rwt`).
- Mayúsculas `S` y `T` significan que el bit especial está puesto pero **falta** el `x` correspondiente: normalmente es un error de configuración.
- Detalle de `chmod`: con notación octal de tres dígitos, `chmod 770` sobre un directorio **no quita** setuid/setgid ya puestos (GNU los preserva); para quitarlos usar `chmod g-s` o cinco dígitos (`chmod 00770`).

**ACL (listas de control de acceso).** El modelo ugo asigna un dueño, un grupo y "todos los demás". Cuando hace falta que **otro** grupo (auditoría) o **un** usuario suelto tenga un permiso distinto, se usan ACL. XFS y ext4 las soportan de fábrica en RHEL 9 (paquete `acl`).
- `getfacl archivo`: ver. `setfacl -m u:ana:rw archivo` (usuario), `setfacl -m g:auditoria:rx dir` (grupo), `-x u:ana` (quitar una entrada), `-b` (quitar todas), `-R` (recursivo), `-d -m ...` (ACL **por defecto** en un directorio: la heredan los archivos y subdirectorios nuevos).
- `ls -l` muestra `+` en lugar de `.` cuando hay ACL. Siempre confirmar con `getfacl`.
- La **`mask`** es el techo de permisos para el grupo dueño y todas las entradas ACL. `chmod` sobre la terna de grupo modifica la `mask`, no el grupo: un `chmod 600` deja las ACL escritas pero sin efecto (`#effective:---`).
- Una ACL en un archivo no sirve si el usuario no puede **atravesar** el directorio: hay que darle al menos `x` en el camino.
- Las ACL se pierden con herramientas que no las conocen: usar `cp -p`/`cp -a`, `rsync -A`, `tar --acls`.

### Lab 4.1 — Permisos estándar y umask (15 min)

- **Objetivo:** leer y modificar permisos con `chmod` (simbólico y octal), entender el efecto de `r`, `w`, `x` en directorios, comprobar el `umask` real y practicar `chown`/`chgrp`. Se trabaja como `student` (root no serviría: no ve los "Permission denied").

1. Crear el material y leer los permisos con los que nace:

```bash
mkdir ~/permisos-lab && cd ~/permisos-lab
echo "informe confidencial de sistemas" > informe.txt
printf '#!/bin/bash\necho "Hola desde $(hostname), soy $(whoami)"\n' > hola.sh
mkdir privado
umask; umask -S
ls -l
```
Salida esperada:
```text
0002
u=rwx,g=rwx,o=rx
total 8
-rw-rw-r--. 1 student student 57 Sep  3 11:05 hola.sh
-rw-rw-r--. 1 student student 33 Sep  3 11:05 informe.txt
drwxrwxr-x. 2 student student  6 Sep  3 11:05 privado
```
Qué observar: `umask 0002` (grupo privado) explica el `664`/`775`. Si un participante ve `0022`, es porque su usuario no tiene grupo privado o su `.bashrc` lo cambia: comprobar con `id -gn` e `id -un`. El script **no** es ejecutable aunque tenga shebang: bash no crea archivos con `x`.

2. Ejecutar un script requiere `x`:

```bash
./hola.sh
chmod u+x hola.sh
ls -l hola.sh
./hola.sh
chmod 750 hola.sh
ls -l hola.sh
```
Salida esperada:
```text
bash: ./hola.sh: Permission denied
-rwxrw-r--. 1 student student 57 Sep  3 11:05 hola.sh
Hola desde rhel01, soy student
-rwxr-x---. 1 student student 57 Sep  3 11:05 hola.sh
```

3. Simbólico y octal sobre el informe (leer `ls -l` tras cada cambio):

```bash
chmod 600 informe.txt;        ls -l informe.txt
chmod o=r informe.txt;        ls -l informe.txt
chmod g+rw informe.txt;       ls -l informe.txt
chmod a-w informe.txt;        ls -l informe.txt
chmod u=rw,g=r,o= informe.txt; ls -l informe.txt
```
Salida esperada:
```text
-rw-------. 1 student student 33 Sep  3 11:05 informe.txt
-rw----r--. 1 student student 33 Sep  3 11:05 informe.txt
-rw-rw-r--. 1 student student 33 Sep  3 11:05 informe.txt
-r--r--r--. 1 student student 33 Sep  3 11:05 informe.txt
-rw-r-----. 1 student student 33 Sep  3 11:05 informe.txt
```
Qué observar: el último equivale a `chmod 640`. Preguntar a la clase el octal de cada línea antes de mostrarlo.

4. `r`, `w` y `x` en un directorio:

```bash
touch privado/a.txt privado/b.txt
chmod 644 privado;  ls privado;  ls -l privado;  cat privado/a.txt
chmod 711 privado;  ls privado;  cat privado/a.txt && echo "cat OK"
chmod 555 privado;  touch privado/c.txt;  rm privado/a.txt
chmod 755 privado
```
Salida esperada (resumida):
```text
a.txt  b.txt
ls: cannot access 'privado/a.txt': Permission denied
ls: cannot access 'privado/b.txt': Permission denied
total 0
-????????? ? ? ? ?            ? a.txt
-????????? ? ? ? ?            ? b.txt
cat: privado/a.txt: Permission denied
ls: cannot open directory 'privado': Permission denied
cat OK
touch: cannot touch 'privado/c.txt': Permission denied
rm: cannot remove 'privado/a.txt': Permission denied
```
Qué observar: con `r` sin `x` se ven los nombres pero nada más (los `?`). Con `x` sin `r` no se puede listar, pero sí usar un archivo cuyo nombre se conoce. Sin `w` en el directorio no se crea ni se borra, aunque `a.txt` sea `664` del propio student. Detalle: como en RHEL `ls` está aliasado a `ls --color=auto`, el `ls privado` "simple" también intenta hacer `stat` de cada entrada para colorearla, así que puede imprimir los mismos `cannot access` que `ls -l`; el orden exacto de esas líneas varía porque salen por la salida de error.

5. El home protege todo: pedro no llega ni a un archivo `644` de student.

```bash
chmod 644 informe.txt
ls -ld ~
sudo -u pedro cat /home/student/permisos-lab/informe.txt
```
Salida esperada:
```text
drwx------. 4 student student 118 Sep  3 11:05 /home/student
cat: /home/student/permisos-lab/informe.txt: Permission denied
```

6. Probar otro `umask` en una subshell (el paréntesis evita cambiar la sesión) y ver el de root:

```bash
(umask 077; touch secreto.txt; mkdir carpeta-secreta)
ls -ld secreto.txt carpeta-secreta
sudo -i umask
```
Salida esperada:
```text
drwx------. 2 student student 6 Sep  3 11:12 carpeta-secreta
-rw-------. 1 student student 0 Sep  3 11:12 secreto.txt
0022
```
Qué observar: root usa `0022`: los archivos que root crea son legibles por todos. Cuando se cree un archivo con datos sensibles como root, aplicar `chmod 600` explícitamente.

7. Dueño y grupo:

```bash
chgrp wheel informe.txt;    ls -l informe.txt
chgrp sistemas informe.txt
sudo chown ana:sistemas informe.txt;  ls -l informe.txt
chown student informe.txt
sudo chown -R student:student ~/permisos-lab
ls -l informe.txt
```
Salida esperada:
```text
-rw-r--r--. 1 student wheel 33 Sep  3 11:05 informe.txt
chgrp: changing group of 'informe.txt': Operation not permitted
-rw-r--r--. 1 ana sistemas 33 Sep  3 11:05 informe.txt
chown: changing ownership of 'informe.txt': Operation not permitted
-rw-r--r--. 1 student student 33 Sep  3 11:05 informe.txt
```
Qué observar: student pudo pasar el archivo a `wheel` (es miembro) pero no a `sistemas`; y una vez que el archivo es de ana, student ya no puede ni recuperarlo: hace falta root. "Operation not permitted" (EPERM) es "esto solo lo hace root o el dueño"; "Permission denied" (EACCES) es "los bits rwx no lo permiten".

- **Checkpoint:** pegar la salida de:
```bash
umask; ls -l ~/permisos-lab
```
Salida esperada: `0002` y, entre otros, `-rwxr-x---. 1 student student ... hola.sh`, `-rw-r--r--. 1 student student ... informe.txt`, `drwx------. 2 student student ... carpeta-secreta`.

### Lab 4.2 — Carpetas compartidas en /srv: setgid, sticky y ACL (25 min)

- **Objetivo:** montar la estructura institucional: `/srv/sistemas` y `/srv/soporte` privadas para su grupo con herencia (setgid), `/srv/publico` como buzón común con sticky bit, y lectura de auditoría vía ACL sin tocar las membresías. Demostrar cada permiso permitido y denegado con `su -`.

1. Crear la estructura:

```bash
sudo mkdir -p /srv/sistemas /srv/soporte /srv/publico
sudo chown root:sistemas /srv/sistemas
sudo chown root:soporte /srv/soporte
sudo chmod 2770 /srv/sistemas /srv/soporte
sudo chmod 1777 /srv/publico
ls -ld /srv/*
```
Salida esperada:
```text
drwxrwxrwt. 2 root root     6 Sep  3 11:20 /srv/publico
drwxrws---. 2 root sistemas 6 Sep  3 11:20 /srv/sistemas
drwxrws---. 2 root soporte  6 Sep  3 11:20 /srv/soporte
```
Qué observar: la `s` en la terna de grupo (setgid) y la `t` en la de otros (sticky). El tamaño `6` es un directorio XFS vacío.

2. ana (sistemas) trabaja en su carpeta y en el buzón público:

```bash
sudo su - ana
```
Como ana:
```bash
cd /srv/sistemas
echo "Plan de mantenimiento Q4" > plan.txt
mkdir proyectos
ls -l
echo "aviso general de ana" > /srv/publico/nota-ana.txt
ls -l /srv/publico
exit
```
Salida esperada:
```text
total 4
-rw-rw-r--. 1 ana sistemas 25 Sep  3 11:22 plan.txt
drwxrwsr-x. 2 ana sistemas  6 Sep  3 11:22 proyectos
total 4
-rw-rw-r--. 1 ana ana 21 Sep  3 11:22 nota-ana.txt
```
Qué observar: en `/srv/sistemas` todo nace con grupo `sistemas` (herencia por setgid) y el subdirectorio hereda además la `s`. En `/srv/publico`, sin setgid, el archivo nace con el grupo primario de ana. Es la misma persona con el mismo `umask`: la diferencia la pone el directorio.

3. carlos (mismo grupo) colabora; pedro (otro grupo) no entra:

```bash
sudo -u carlos bash -c 'echo "Revisado por carlos" >> /srv/sistemas/plan.txt && cat /srv/sistemas/plan.txt'
sudo -u pedro ls /srv/sistemas
sudo -u pedro cat /srv/sistemas/plan.txt
```
Salida esperada:
```text
Plan de mantenimiento Q4
Revisado por carlos
ls: cannot open directory '/srv/sistemas': Permission denied
cat: /srv/sistemas/plan.txt: Permission denied
```
Qué observar: `plan.txt` es `664` (otros pueden leer), pero pedro no puede **atravesar** `/srv/sistemas` (`---` para otros). El directorio manda.

4. Sticky bit en `/srv/publico`: pedro lee la nota de ana, deja la suya, pero no puede borrar la ajena:

```bash
sudo su - pedro
```
Como pedro:
```bash
cat /srv/publico/nota-ana.txt
echo "nota de pedro" > /srv/publico/nota-pedro.txt
rm /srv/publico/nota-ana.txt
rm /srv/publico/nota-pedro.txt && echo "borre la mia"
exit
```
Salida esperada (responder `y` a la pregunta de `rm`):
```text
aviso general de ana
rm: remove write-protected regular file '/srv/publico/nota-ana.txt'? y
rm: cannot remove '/srv/publico/nota-ana.txt': Operation not permitted
borre la mia
```
Qué observar: `rm` primero pregunta porque el archivo no es escribible por pedro (con `rm -f` no preguntaría); al confirmar, el directorio es `777` (pedro tiene `w`) y aun así el kernel responde EPERM por el sticky bit. Sin sticky (`sudo chmod o-t /srv/publico`) pedro podría borrar lo de ana. Mismo comportamiento que `/tmp`.

5. Bits especiales que ya existen en el sistema:

```bash
ls -l /usr/bin/passwd /usr/bin/su /etc/shadow
ls -ld /tmp
sudo find / -xdev -type f -perm -4000 2>/dev/null | head -8
sudo find / -xdev -type f -perm -4000 2>/dev/null | wc -l
sudo find / -xdev -type f -perm -2000 2>/dev/null
touch ~/permisos-lab/raro; chmod 4644 ~/permisos-lab/raro; ls -l ~/permisos-lab/raro; rm ~/permisos-lab/raro
```
Salida esperada (resumida; la lista exacta varía):
```text
----------. 1 root root  1783 Sep  3 09:32 /etc/shadow
-rwsr-xr-x. 1 root root 32648 Feb 12  2025 /usr/bin/passwd
-rwsr-xr-x. 1 root root 57536 Feb 12  2025 /usr/bin/su
drwxrwxrwt. 12 root root 4096 Sep  3 11:25 /tmp
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/su
/usr/bin/umount
/usr/bin/passwd
/usr/bin/sudo
25
/usr/bin/write
/usr/libexec/utempter/utempter
-rwSr--r--. 1 student student 0 Sep  3 11:26 /home/student/permisos-lab/raro
```
Qué observar: el número final (aquí `25`) **depende de los paquetes instalados** —en una instalación "Server" mínima suele estar entre 15 y 25—; no hay que memorizarlo, lo que importa es saber sacar el inventario y compararlo con el de ayer. `passwd` (setuid root) es lo que permite a ana modificar `/etc/shadow` (modo `000`). Casi todos los comandos de hoy (`su`, `sudo`, `chage`, `gpasswd`, `newgrp`) son setuid: por eso deben venir de paquetes firmados. La `S` mayúscula del archivo `raro` indica setuid sin `x`: configuración inválida.

6. Auditoría con **solo lectura vía ACL**. laura no es de `sistemas` ni de `soporte` y no debe serlo (sería escritura). Antes:

```bash
sudo -u laura ls /srv/sistemas
```
Salida esperada:
```text
ls: cannot open directory '/srv/sistemas': Permission denied
```
Aplicar la ACL de forma recursiva a ambas carpetas y verificar:

```bash
sudo setfacl -R -m g:auditoria:rX /srv/sistemas /srv/soporte
ls -ld /srv/sistemas /srv/soporte
getfacl /srv/sistemas
```
Salida esperada:
```text
drwxrws---+ 2 root sistemas 39 Sep  3 11:22 /srv/sistemas
drwxrws---+ 2 root soporte   6 Sep  3 11:20 /srv/soporte
getfacl: Removing leading '/' from absolute path names
# file: srv/sistemas
# owner: root
# group: sistemas
# flags: -s-
user::rwx
group::rwx
group:auditoria:r-x
mask::rwx
other::---
```
Qué observar: el `.` se convirtió en `+`. `flags: -s-` es el setgid. La `X` mayúscula dio `x` al directorio y **no** a `plan.txt` (que recibió `r--`).

```bash
sudo -u laura ls -l /srv/sistemas
sudo -u laura cat /srv/sistemas/plan.txt
sudo -u laura touch /srv/sistemas/intruso.txt
sudo getfacl /srv/sistemas/plan.txt | grep auditoria
```
Salida esperada:
```text
total 4
-rw-rw-r--+ 1 ana sistemas 45 Sep  3 11:24 plan.txt
drwxrwsr-x+ 2 ana sistemas  6 Sep  3 11:22 proyectos
Plan de mantenimiento Q4
Revisado por carlos
touch: cannot touch '/srv/sistemas/intruso.txt': Permission denied
group:auditoria:r--
```

7. El problema de los archivos **nuevos** y la ACL por defecto. ana crea un archivo confidencial y le quita la lectura a otros:

```bash
sudo -u ana bash -c 'echo "claves del respaldo" > /srv/sistemas/confidencial.txt; chmod 660 /srv/sistemas/confidencial.txt'
sudo ls -l /srv/sistemas/confidencial.txt
sudo -u laura cat /srv/sistemas/confidencial.txt
```
Salida esperada:
```text
-rw-rw----. 1 ana sistemas 20 Sep  3 11:30 /srv/sistemas/confidencial.txt
cat: /srv/sistemas/confidencial.txt: Permission denied
```
Qué observar: la ACL del Paso 6 se aplicó a lo que existía **en ese momento**; el archivo nuevo nació sin ella (`.` en vez de `+`). Nótese que `student` necesita `sudo` incluso para hacer `ls -l` de un archivo dentro de `/srv/sistemas`: no es miembro de `sistemas` ni tiene ACL, así que no atraviesa el directorio (`---` para otros). Solución: ACL por defecto en el directorio, más una pasada recursiva para lo ya existente:

```bash
sudo setfacl -d -m g:auditoria:rx /srv/sistemas /srv/soporte /srv/sistemas/proyectos
sudo setfacl -R -m g:auditoria:rX /srv/sistemas /srv/soporte
getfacl /srv/sistemas | grep default
sudo -u ana bash -c 'echo "otro secreto" > /srv/sistemas/nuevo.txt; chmod 660 /srv/sistemas/nuevo.txt'
sudo ls -l /srv/sistemas/nuevo.txt
sudo getfacl /srv/sistemas/nuevo.txt | grep -E 'auditoria|mask'
sudo -u laura cat /srv/sistemas/nuevo.txt
```
Salida esperada:
```text
default:user::rwx
default:group::rwx
default:group:auditoria:r-x
default:mask::rwx
default:other::---
-rw-rw----+ 1 ana sistemas 13 Sep  3 11:33 /srv/sistemas/nuevo.txt
group:auditoria:r-x                     #effective:r--
mask::rw-
otro secreto
```
Qué observar: `nuevo.txt` nació con `+`: heredó la entrada de auditoría. El `chmod 660` de ana fijó la `mask` en `rw-`, que sigue permitiendo `r`. Si ana hiciera `chmod 600`, la `mask` pasaría a `---` y laura vería `#effective:---` y "Permission denied": las ACL están sujetas al techo de la `mask`.

8. *(Opcional — "Si sobra tiempo"; si quedan menos de 5 min, el instructor lo demuestra.)* ACL de usuario y su límite: pedro recibe lectura sobre `plan.txt`, pero sigue sin poder atravesar el directorio:

```bash
sudo setfacl -m u:pedro:r /srv/sistemas/plan.txt
sudo -u pedro cat /srv/sistemas/plan.txt
sudo setfacl -m u:pedro:x /srv/sistemas
sudo -u pedro cat /srv/sistemas/plan.txt | head -1
sudo -u pedro ls /srv/sistemas
sudo setfacl -x u:pedro /srv/sistemas /srv/sistemas/plan.txt
getfacl /srv/sistemas | grep pedro || echo "pedro sin ACL"
```
Salida esperada:
```text
cat: /srv/sistemas/plan.txt: Permission denied
Plan de mantenimiento Q4
ls: cannot open directory '/srv/sistemas': Permission denied
pedro sin ACL
```
Qué observar: con `x` en el directorio (y sin `r`) pedro accede a lo que conoce por nombre, pero no puede listar. `-x` quita una entrada concreta; `-b` las quitaría todas (`sudo setfacl -b archivo`).

- **Checkpoint:** pegar la salida de:
```bash
ls -ld /srv/*; getfacl /srv/sistemas 2>/dev/null | grep -v '^#'
```

## Reto individual (30 min)

**Ticket #2026-0312 — Alta del proyecto "Expediente Digital"**

> Solicitante: Coordinación de Proyectos. Servidor: `rhel01`.
>
> 1. Crear el grupo `expediente` (GID 3010) y los usuarios `maria` (UID 2010) y `jorge` (UID 2011), ambos con `expediente` como grupo suplementario, contraseña inicial `Pgn.2026`, contraseña que caduca cada 60 días con aviso 10 días antes, y **cuenta** que expira el 2026-12-31 (contrato por proyecto).
> 2. Crear la carpeta compartida `/srv/expediente`: dueño `root`, grupo `expediente`; solo los miembros del grupo pueden entrar, leer y escribir; todo archivo o subcarpeta nuevo debe pertenecer al grupo `expediente` automáticamente. Nadie más entra.
> 3. Crear el usuario `auditor_ext` (UID 2012, contraseña `Pgn.2026`), que **no** debe ser miembro de `expediente`, con permiso de **solo lectura** sobre `/srv/expediente` y sobre todo lo que se cree dentro en el futuro.
> 4. Los miembros de `expediente` deben poder ejecutar como root **únicamente** `systemctl status chronyd` y `systemctl restart chronyd`. La regla va en su propio archivo bajo `/etc/sudoers.d/`. `laura` sigue sin ningún privilegio.

**Verificación que debe entregar el participante** (pegar en el chat la salida de estos comandos):

```bash
id maria; id jorge; id auditor_ext
sudo chage -l maria | grep -E 'Account expires|Maximum|warning'
ls -ld /srv/expediente
sudo -u maria bash -c 'echo caso > /srv/expediente/caso-001.txt'; sudo ls -l /srv/expediente
sudo -u auditor_ext cat /srv/expediente/caso-001.txt
sudo -u auditor_ext touch /srv/expediente/x.txt
sudo -u laura ls /srv/expediente
sudo -l -U jorge | tail -1
sudo -l -U laura
sudo visudo -c
```

Resultado esperado: `maria` y `jorge` con `groups=...,3010(expediente)`; `auditor_ext` sin `expediente`; cuenta expira `Dec 31, 2026`, máximo 60, aviso 10; `drwxrws---+ 2 root expediente ... /srv/expediente`; `caso-001.txt` con grupo `expediente`; `auditor_ext` lee (`caso`) pero recibe `Permission denied` al crear; `laura` recibe `Permission denied`; `jorge` puede `(root) /usr/bin/systemctl status chronyd, /usr/bin/systemctl restart chronyd`; `laura` "is not allowed to run sudo"; `parsed OK`.

### Solución (para el instructor)

```bash
# 1. Grupo, usuarios, contraseñas, caducidad
sudo groupadd -g 3010 expediente
sudo useradd -m -u 2010 -G expediente -c "Maria Perez - Expediente" -e 2026-12-31 maria
sudo useradd -m -u 2011 -G expediente -c "Jorge Diaz - Expediente"  -e 2026-12-31 jorge
sudo useradd -m -u 2012 -c "Auditor externo" auditor_ext
for u in maria jorge auditor_ext; do echo 'Pgn.2026' | sudo passwd --stdin $u; done
sudo chage -M 60 -W 10 maria
sudo chage -M 60 -W 10 jorge
# (equivalente a -e: sudo chage -E 2026-12-31 maria)

# 2. Carpeta compartida con herencia de grupo
sudo mkdir -p /srv/expediente
sudo chown root:expediente /srv/expediente
sudo chmod 2770 /srv/expediente

# 3. Solo lectura para auditor_ext, presente y futuro
sudo setfacl -m u:auditor_ext:rx /srv/expediente
sudo setfacl -d -m u:auditor_ext:rx /srv/expediente

# 4. Regla sudo limitada
sudo visudo -f /etc/sudoers.d/expediente
```
Contenido de `/etc/sudoers.d/expediente`:
```text
%expediente ALL=(root) /usr/bin/systemctl status chronyd, /usr/bin/systemctl restart chronyd
```
Errores que se verán al corregir: olvidar el `-d` en la ACL (el auditor lee lo existente pero no lo nuevo), usar `chmod 770` sin el `2` (los archivos nacen con el grupo privado de maria), poner `/bin/systemctl` en vez de `/usr/bin/systemctl` (funciona por el enlace `/bin -> /usr/bin`, pero `sudo -l` muestra la ruta escrita; aceptar ambas), y agregar a `auditor_ext` al grupo (viola el requisito: tendría escritura).

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `id [usuario]` | UID, GID y grupos de un usuario (sin argumento: el propio) |
| `getent passwd usuario` / `getent group grupo` | Consultar cuentas y grupos vía NSS (locales, LDAP, AD) |
| `who`, `w`, `last -n 5`, `sudo lastb`, `lastlog` | Sesiones activas, historial de ingresos, intentos fallidos, último ingreso por cuenta |
| `sudo useradd -m -u UID -c "Nombre" -G grupo -s /bin/bash usuario` | Crear usuario con UID, comentario, grupo suplementario y shell |
| `sudo useradd -r -s /sbin/nologin svc` | Cuenta de sistema (UID < 1000) sin acceso interactivo |
| `useradd -D`; `/etc/default/useradd`; `/etc/login.defs`; `/etc/skel` | Valores por defecto de las cuentas nuevas |
| `sudo passwd usuario`; `echo 'clave' \| sudo passwd --stdin usuario` | Asignar contraseña (interactivo / desde script) |
| `sudo passwd -S usuario` | Estado: `PS` con contraseña, `LK` bloqueada, `NP` sin contraseña |
| `sudo passwd -l/-u usuario`; `sudo usermod -L/-U usuario` | Bloquear / desbloquear la contraseña |
| `sudo usermod -aG grupo usuario` | Añadir a un grupo suplementario (**siempre con `-a`**) |
| `sudo usermod -s /sbin/nologin usuario` | Quitar la shell interactiva |
| `sudo usermod -e AAAA-MM-DD usuario`; `sudo chage -E 0 usuario` | Expirar la cuenta (bloqueo total); `chage -E -1` la reactiva |
| `sudo chage -l usuario` | Ver política de contraseña y expiración |
| `sudo chage -M 90 -m 1 -W 7 usuario` | Máximo 90 días, mínimo 1, aviso 7 días antes |
| `sudo chage -d 0 usuario` | Obligar a cambiar la contraseña en el próximo ingreso |
| `sudo userdel -r usuario`; `sudo find / -xdev -nouser` | Borrar cuenta con home; buscar archivos huérfanos |
| `sudo groupadd -g GID grupo`; `sudo groupmod -n nuevo viejo`; `sudo groupdel grupo` | Crear, renombrar, borrar grupos |
| `sudo gpasswd -a/-d usuario grupo`; `sudo gpasswd -A usuario grupo` | Añadir/quitar miembro; nombrar administrador del grupo |
| `newgrp grupo` | Shell con otro grupo primario temporal |
| `su - usuario`; `su -` | Shell de login como otro usuario / como root (pide la clave del destino) |
| `sudo -i`; `sudo -u usuario comando`; `sudo -l`; `sudo -k` | Shell root con clave propia; ejecutar como otro; listar privilegios; olvidar credencial |
| `sudo visudo`; `sudo visudo -f /etc/sudoers.d/archivo`; `sudo visudo -c` | Editar `sudoers` con validación; archivo de reglas propio; comprobar sintaxis |
| `%grupo ALL=(root) /usr/bin/cmd arg` | Sintaxis de regla: quién, dónde, como quién, qué |
| `sudo grep 'sudo\[' /var/log/secure`; `sudo journalctl -t sudo` | Auditar el uso de sudo |
| `chmod 2770 dir`; `chmod 1777 dir`; `chmod u+x,g-w,o= archivo` | Permisos octal (con bit especial) y simbólico |
| `sudo chown usuario:grupo -R ruta`; `chgrp grupo archivo` | Cambiar dueño y grupo |
| `umask`; `umask -S`; `umask 077` | Ver / fijar la máscara de creación |
| `sudo find / -xdev -type f -perm -4000` | Inventario de binarios setuid (`-2000` setgid) |
| `getfacl ruta`; `sudo setfacl -m u:ana:rw archivo`; `-m g:grupo:rX -R dir` | Ver y asignar ACL a usuario o grupo, recursivo |
| `sudo setfacl -d -m g:grupo:rx dir` | ACL por defecto: la heredan los archivos nuevos |
| `sudo setfacl -x u:ana ruta`; `sudo setfacl -b ruta` | Quitar una entrada / todas las ACL |

## Notas para el instructor

- **Preparar antes de la clase:**
  - Restaurar `dia02-fin` en la VM del instructor y ejecutar **todos** los labs del día en orden, cronometrando. El Lab 4.2 (ACL) es el que más se desvía del tiempo.
  - Tener `~/crear-usuarios.sh` probado y a mano para reconstruir usuarios si alguien restaura un snapshot a mitad de clase.
  - Preparar el mensaje de `visudo` con error: escribir a propósito `%soporte ALL=(root) /usr/bin/systemctl restart chronyd,` (coma final) y mostrar el `What now?`. Practicar la recuperación de un `sudoers.d` roto con `su -` y borrado del archivo.
  - Verificar `umask` de `student` y de `root` en la VM (`0002` / `0022`) y revisar `/etc/bashrc` para poder explicarlo si alguien pregunta. ⚠️ Verificar en la VM antes de la clase: que dentro de `newgrp sistemas` el `umask` pasa a `0022` (Lab 2.1, paso 10) y qué prefijo exacto deja `passwd -l` en `/etc/shadow` (`!!` con `passwd -l`, `!` con `usermod -L`; Lab 2.2, paso 5). ⚠️ Verificar también: que `visudo` respeta `$EDITOR` dentro de `sudo -i` (Lab 3.1, paso 5), y el conteo real de binarios setuid de la VM (Lab 4.2, paso 5), para no anunciar un número que no coincida con lo que verán los participantes.
  - Comprobar que `chronyd` está activo (`systemctl is-active chronyd`): es el servicio que reinicia `soporte`.
  - Tomar snapshot `dia03-inicio` antes de empezar; `dia03-fin` al terminar.
  - Tener lista la respuesta a "¿se puede integrar con Active Directory?" (sí: `realmd`/`sssd`; fuera del alcance del curso; `getent` e `id` funcionan igual).

- **Qué estudiar si es nuevo en RHEL:**
  1. Campos de `/etc/shadow` y `chage`: `man 5 shadow`, `man chage`. Practicar `chage -d 0` y ver el flujo de cambio forzado con `su - usuario`.
  2. Sintaxis de `sudoers`: `man 5 sudoers`, secciones "User Specification", "Aliases" y "Command matching" (argumentos exactos, comodines). Practicar `sudo -l -U usuario` y leer las líneas de `/var/log/secure`.
  3. Semántica de la `mask` en ACL: `man 5 acl` (sección "Access check algorithm") y `man setfacl`. Reproducir el paso 7 del Lab 4.2 con `chmod 600` para ver `#effective:---`.
  4. Preservación de setgid en `chmod` octal: `info coreutils 'Directory Setuid and Setgid'` o la sección "SETUID AND SETGID BITS" de `man chmod`. Probar `chmod 770` vs `chmod 00770` vs `chmod g-s` sobre un directorio `2770`.
  5. Defaults de RHEL 9 que difieren de otras distribuciones: `HOME_MODE 0700`, `USERGROUPS_ENAB yes`, `umask` condicional en `/etc/bashrc`, `passwd --stdin`, `/sbin/nologin`. Leer `/etc/login.defs` completo (son 30 líneas activas).

- **Errores frecuentes de los participantes y cómo resolverlos:**

| Síntoma | Causa | Solución |
|---|---|---|
| `useradd: group 'sistemas' does not exist` | Creó el usuario antes que el grupo | `groupadd` primero; luego `usermod -aG sistemas usuario` |
| `useradd: UID 2001 is not unique` / `user 'ana' already exists` | Ya existía (snapshot o reintento) | `id ana`; si es correcto seguir; si no, `userdel -r ana` y repetir |
| Un usuario perdió acceso a su carpeta de área tras un cambio de grupos | `usermod -G` sin `-a` reemplazó la lista | `usermod -aG grupo usuario` (o `gpasswd -a`); repasar con `id` |
| `id` muestra el grupo nuevo pero el usuario "no puede entrar" a la carpeta | Sesión abierta con la lista de grupos vieja | Cerrar sesión y volver a entrar (o `newgrp grupo`) |
| `su: Authentication failure` al probar un usuario | Contraseña incorrecta, o cuenta bloqueada con `passwd -l` | `sudo passwd -S usuario`; si `LK`, `passwd -u`; si duda, reasignar clave |
| `This account is currently not available.` | Shell `/sbin/nologin` | `usermod -s /bin/bash usuario` si debe tener acceso |
| `visudo` muestra `syntax error` y `What now?` | Coma final, alias mal escrito, ruta sin `/` inicial | Pulsar `e`, corregir; nunca `Q` |
| `sudo: /etc/sudoers.d/x: syntax error` y **nadie** puede usar sudo | Se guardó un archivo roto (con `Q`, o editado sin `visudo`) | `su -` con clave de root; `rm /etc/sudoers.d/x` o `visudo -f` para corregir |
| `sudo: a password is required` dentro de `sudo -u pedro ...` | Está anidando `sudo` dentro de otro `sudo` | Usar `su - pedro` y luego `sudo`, o `sudo -l -U pedro` |
| `Sorry, user pedro is not allowed to execute ... --no-pager` | La regla fija los argumentos; el comando lleva argumentos distintos | Ejecutar exactamente el comando de la regla o ampliar la regla |
| La carpeta `2770` no hereda el grupo | Hizo `chmod 770` (perdió la `s`) o `chown` posterior a un grupo distinto | `chmod 2770`; verificar con `ls -ld` que aparece `rws` |
| `chmod 770` no quita la `s` del directorio | GNU chmod preserva setuid/setgid con octal de 3 dígitos | `chmod g-s dir` o `chmod 00770 dir` |
| `setfacl: Operation not supported` | Sistema de archivos sin ACL (raro en RHEL 9; ocurre en algunos montajes de red) | Trabajar bajo `/srv` (XFS raíz); verificar con `findmnt -T /srv` |
| laura lee los archivos viejos pero no los nuevos | Falta la ACL por defecto (`-d`) | `setfacl -d -m g:auditoria:rx dir` y `setfacl -R -m ...` para lo existente |
| laura tiene ACL `r` en el archivo y aun así "Permission denied" | No puede atravesar el directorio, o la `mask` la limita (`#effective:---`) | Dar `x` en el directorio; revisar `getfacl` y `chmod` reciente |
| `rm: ... Operation not permitted` en `/srv/publico` | Sticky bit: solo el dueño borra | Es el comportamiento correcto; borrar como dueño o root |
| `chown: Operation not permitted` como student | Solo root cambia el dueño | `sudo chown` |
| `ls: cannot access '/srv/sistemas/...': Permission denied` al ejecutar `ls -l` o `getfacl` como student | student no es de `sistemas`; no atraviesa un directorio `2770` | Anteponer `sudo` (o usar `sudo -u ana`) |
| `rm` se queda esperando (`remove write-protected regular file?`) | El archivo no es escribible por quien borra; `rm` pregunta antes | Responder `y`, o usar `rm -f`; el sticky bit responde después |

- **Diferencias VirtualBox (x86_64) vs UTM (aarch64):** no aplican a este día. Todo es independiente del hipervisor. Únicos detalles visibles: `who`/`last` muestran la IP `10.0.2.2` (puerta de enlace NAT) en ambos; los tamaños de los binarios setuid en `ls -l` difieren entre arquitecturas (irrelevante).

- **Preguntas probables y respuesta corta:**
  - *¿Por qué no trabajar siempre como root si igual tengo sudo?* Trazabilidad (el log dice quién) y contención de errores (un `rm -rf` mal escrito como student borra menos). El RHCSA y cualquier auditoría lo asumen.
  - *¿Cómo doy sudo total a un compañero?* `sudo usermod -aG wheel usuario`. Y para quitárselo: `sudo gpasswd -d usuario wheel`.
  - *¿Por qué `id` muestra el grupo nuevo pero el usuario no tiene acceso?* Los grupos se cargan al iniciar sesión; debe volver a entrar.
  - *¿`passwd -l`, `nologin` o `chage -E 0`?* Bloqueo de contraseña (llave SSH sigue entrando), sin shell (otros servicios siguen), expiración de cuenta (todo cerrado). Para desvinculaciones: expirar.
  - *¿Puedo exigir complejidad de contraseñas?* Sí, en `/etc/security/pwquality.conf` (`minlen`, `dcredit`, `ucredit`...). Aplica a usuarios que cambian su clave, no a root. Fuera del alcance del RHCSA, pero útil en la institución.
  - *¿Qué diferencia hay entre "Permission denied" y "Operation not permitted"?* EACCES: los bits rwx/ACL no lo permiten. EPERM: la operación está reservada a root o al dueño (chown, sticky, setuid).
  - *¿Por qué hay tantos usuarios con `nologin` en `/etc/passwd`?* Son cuentas de servicio: cada demonio corre con su propio UID para que un fallo en uno no comprometa a los demás. Nunca borrarlas.
  - *¿setuid en un script sirve?* No, el kernel lo ignora por seguridad. Se usa `sudo` con una regla.
  - *¿ACL o grupos?* Grupos para lo estructural (áreas); ACL para excepciones (un auditor, un contratista). Si un directorio necesita cinco ACL, probablemente falta un grupo.
  - *¿Por qué `getfacl` quita la `/` inicial?* Para que la salida pueda usarse con `setfacl --restore` desde cualquier directorio. Es solo un aviso.
  - *¿Qué pasa con los archivos de un usuario borrado?* Quedan con UID numérico; si se reutiliza el UID cambian de dueño. Buscar con `find -nouser` antes y después.
  - *¿Cómo veo quién usó sudo ayer?* `sudo journalctl -t sudo --since yesterday --until today` o `sudo grep 'sudo\[' /var/log/secure*`.

- **Relación con el examen RHCSA (EX200):** este día cubre completo el objetivo "Manage users and groups": crear, borrar y modificar cuentas locales; cambiar contraseñas y ajustar caducidad (`chage`); crear, borrar y modificar grupos y membresías; **configurar acceso de superusuario** (`sudoers`, grupo `wheel`, reglas en `sudoers.d`). Del objetivo "Manage files from the command line / Manage security": listar, fijar y cambiar permisos ugo/rwx; **crear y configurar directorios set-GID para colaboración**; **crear y administrar ACL**; diagnosticar y corregir problemas de permisos. Formato típico del examen: "Crear el usuario natasha con UID 3500 y shell no interactiva; el usuario harry debe pertenecer al grupo admin como suplementario; todos con contraseña X" y "El directorio /home/admins pertenece al grupo admin, sus miembros pueden leer y escribir, los demás no, y los archivos creados dentro pertenecen al grupo admin". Se resuelve en minutos con lo practicado hoy; la trampa habitual es olvidar `-a` en `usermod` o el `2` de `2770`.

## Tarea y preparación para el día siguiente

1. **Snapshot `dia03-fin`** con los usuarios, grupos, `/srv/*`, la regla de `soporte` y el reto resuelto. Estos usuarios y carpetas se reutilizan el Día 10 (tickets de troubleshooting): no borrarlos.
2. **Verificar la suscripción**, porque el Día 04 se instala `httpd`: `sudo subscription-manager status` debe decir `Overall Status: Registered` (en versiones anteriores de `subscription-manager` dice `Disabled` con la nota `Content Access Mode is set to Simple Content Access`; también es válido) y `dnf repolist` debe listar `rhel-9-for-<arch>-baseos-rpms` y `appstream`. Quien vea `This system has no repositories available` debe registrar la VM antes de la clase con `sudo subscription-manager register` (usuario y clave de developers.redhat.com).
3. **Práctica de 20 minutos** (sin mirar el material; luego comparar):
   - Crear el usuario `prueba` (UID 2500, grupo suplementario `soporte`, contraseña `Pgn.2026`, caduca a los 45 días), bloquearlo, comprobar con `passwd -S`, desbloquearlo, expirar la cuenta y borrarlo con `-r`.
   - Crear `/srv/prueba` con `2770` para el grupo `soporte`, comprobar la herencia de grupo con `sudo su - pedro`, dar lectura a `laura` por ACL (con ACL por defecto) y comprobarlo.
   - Leer las últimas 10 líneas de sudo en `/var/log/secure` y explicar cada una.
4. **Lectura:** `man 5 sudoers` (solo la sección "EXAMPLES", 5 minutos) y `man chage`.
5. **Para el Día 04** (procesos, servicios y logs): tener a mano `ps`, `top`, `systemctl status sshd` y `journalctl -b`; no hace falta preparar nada más en la VM.
