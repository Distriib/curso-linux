# Día 10 — Troubleshooting, logs avanzados, recuperación del sistema y laboratorio integrador
> Al terminar, el participante diagnostica un servidor con fallas siguiendo un método por capas, lee y correlaciona logs de journald, rsyslog y auditd, recupera un sistema que no arranca (contraseña de root perdida, `/etc/fstab` roto, kernel dañado) y resuelve, sin pistas, ocho tickets reales sobre el servidor `web01` que construyó durante el curso.

**Ficha técnica cubierta:**
- RH134 M8 Troubleshooting: Diagnóstico de fallos, Logs avanzados, Recuperación básica (módulo completo).
- Cierre e integración de los 16 módulos de RH124 y RH134: el laboratorio integrador toca RH124 M3 (usuarios, permisos), M4 (servicios, logs), M5 (red), M7 (almacenamiento), M8 (firewall, SELinux) y RH134 M1/M2 (script y cron), M3 (NetworkManager), M4 (HTTP, NFS), M5 (SELinux, firewalld), M6 (LVM), M7 (Podman).

**Requisitos previos:**
- Snapshot `dia09-fin` tomado. VM `rhel01` encendida y acceso por `ssh -p 2222 student@localhost`.
- **Contraseña de root a la mano** y **acceso a la consola de la VM** (ventana de VirtualBox/UTM): hoy se rompe el arranque a propósito y SSH no sirve en emergency mode ni en el initramfs.
- Suscripción activa (`dnf repolist` lista BaseOS y AppStream): se instala `sysstat`, `strace`, `lsof` y, si falta, `audit`.
- Lo construido en días anteriores y que hoy se reutiliza: grupo `sistemas` (Día 3), `backup.sh` en `~/bin` (Día 7), `/var/log/monitor-disco.log` (Día 7), regla `fcontext` para `/web` (Día 8), `httpd` en el puerto 80 con `00-default.conf` (Día 9), `nfs-server` exportando `/srv/nfs/compartido` (Día 9), contenedor `portal` con Quadlet y linger (Reto Día 9), LV `lv_backups` montado en `/backups` (Reto Día 6), perfil NetworkManager `lab` con IP host-only (Día 5). Si algo falta, el script `construir-web01.sh` del Bloque 4 lo crea.
- Aviso de la tarea del Día 9: debe haber al menos 3 GB libres en `/` (`df -h /`).

---

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:30 | 30 | Bloque 1 — Método de diagnóstico | Repaso Día 9 (3 preguntas). El método en 7 pasos; orden por capas; tabla de herramientas por capa. Lab 1.1: radiografía del servidor en 12 comandos, archivos borrados que ocupan espacio, `sysstat`, OOM killer |
| 0:30–1:10 | 40 | Bloque 2 — Logs avanzados | Lab 2.1 journald a fondo (boots, filtros por campo, `-o verbose/json-pretty`, retención, `journald.conf`). Lab 2.2 rsyslog: reglas, plantillas, envío y recepción TCP 514 con la misma VM. Lab 2.3 logrotate. Lab 2.4 auditd y `/var/log/secure` |
| 1:10–1:40 | 30 | Bloque 3A — Recuperación: arranque y root | Proceso de arranque, GRUB2, `grubby`, targets (10). Lab 3.1: GRUB visible, kernels y targets (5). Lab 3.2: restablecer la contraseña de root con `rd.break`, alternativa `init=/bin/bash` (15) |
| 1:40–1:55 | 15 | Descanso | Las VMs terminan el relabel de SELinux de la Lab 3.2 mientras tanto |
| 1:55–2:10 | 15 | Bloque 3B — Recuperación: fstab | Verificación del relabel (paso 6 del Lab 3.2) (1). Lab 3.3: reparar `/etc/fstab` desde emergency mode (12) + menciones: kernel, `dracut`, Cockpit (2) |
| 2:10–2:30 | 20 | Bloque 4 — Construcción de `web01` | `construir-web01.sh` + `verificar-web01.sh`; snapshot `dia10-pre-romper` |
| 2:30–3:40 | 70 | Reto integrador | `romper.sh`: 8 tickets (3 obligatorios, 2 intermedios, 3 bonus). Evaluación con rúbrica |
| 3:40–4:00 | 20 | Cierre del curso | Puesta en común, mapa de la ficha técnica, objetivos RHCSA por día, cómo seguir, snapshot `dia10-fin` |

---

## Prioridad si falta tiempo

**Imprescindible** (lo que el participante debe salir sabiendo y habiendo practicado):
- El método: síntoma → qué debería pasar → hipótesis → prueba → corrección → verificación → documentación, recorriendo las capas en orden. Saber cuál es el **primer comando** de cada capa (tabla del Bloque 1).
- `systemctl --failed`, `journalctl -p err -b`, `journalctl -u X --since`, `journalctl -xe`, `dmesg -T`, `df -h` + `lsof +L1`, `ss -tulpn`, `firewall-cmd --list-all`, `ausearch -m AVC -ts recent`.
- journald: `--list-boots`, `-b -1`, filtros `_SYSTEMD_UNIT=`/`_PID=`/`_COMM=`, persistencia en `/var/log/journal` y `journald.conf`.
- Restablecer la contraseña de root con `rd.break` y salir de emergency mode por un `/etc/fstab` roto (objetivos RHCSA "Interrupt the boot process in order to gain access to a system" y "Diagnose and correct boot issues").
- Laboratorio integrador: tickets T1, T2 y T3 resueltos y **verificados desde afuera**.

**Importante:**
- rsyslog remoto (cliente y servidor en la misma VM) y `logger -n` para probar.
- logrotate con `-d` y `-f`; auditd con una regla `-w` y `ausearch -k`.
- `grubby` (`--default-kernel`, `--info=ALL`, `--args`/`--remove-args`, `--set-default`), `systemctl set-default`, `rescue.target` vs `emergency.target`.
- Tickets T4 y T5.
- Mapa de la ficha técnica y objetivos RHCSA por día (cierre).

**Si sobra tiempo** (demo del instructor o tarea):
- `sysstat` (`sar`, `iostat`), `strace -p`, `systemd-analyze critical-chain`.
- Plantillas de rsyslog por host, `aureport`, análisis de `/var/log/secure` con `awk`.
- `dracut -f`, elegir otro kernel en GRUB, Cockpit como consola de emergencia.
- Tickets T6, T7 y T8 (T6 puede omitirse con `romper.sh --sin-fstab`).

---

## Bloque 1 — Método de diagnóstico (30 min)

### Conceptos (12 min)

**Repaso del Día 9 (3 preguntas, con la terminal abierta):**
1. ¿Por qué el contenedor `portal` desaparece tras reiniciar si no se hizo `loginctl enable-linger student`? (Los servicios `--user` solo viven mientras el usuario tiene sesión; linger arranca su instancia de systemd al encender.)
2. ¿Cuál es el error más común al escribir `/etc/exports`? (Espacio entre el cliente y las opciones: exporta a todo el mundo con opciones por defecto; `exportfs -v` lo delata.)
3. ¿Qué hace `:Z` en `podman run -v`? (Reetiqueta el directorio con `container_file_t` y una categoría MCS privada del contenedor.)

**El método.** Un ticket nunca dice "el contexto SELinux de `/web/index.html` es `admin_home_t`"; dice "la web da error". El trabajo del administrador es convertir un síntoma en una causa, y eso se hace con un procedimiento, no con intuición. Decir en clase: "Un buen técnico no es el que conoce más comandos, es el que **no se salta pasos** cuando está bajo presión". Los siete pasos, en este orden:

1. **Síntoma.** Qué reporta el usuario, con sus palabras. ¿Desde cuándo? ¿Para todos o para uno? ¿Qué cambió? (`last`, `journalctl --since`, `dnf history`, `sudo grep sudo /var/log/secure`).
2. **¿Qué debería pasar?** Escribir la cadena esperada: "el navegador del usuario llega al puerto 80 de la IP host-only, firewalld deja pasar, httpd escucha, lee `/web/index.html`, SELinux lo permite, responde 200".
3. **Hipótesis.** Una por capa, de la más barata de comprobar a la más cara.
4. **Prueba.** Un comando que confirme o descarte la hipótesis. Si no la confirma, siguiente hipótesis; no "arreglar por si acaso".
5. **Corrección.** Mínima y persistente (`nmcli` y no `ip addr`; `--permanent`; `semanage` y no `chcon`; `enable --now` y no solo `start`).
6. **Verificación.** Desde donde el usuario lo ve: el navegador del equipo propio, no `curl localhost`. Y **tras un reinicio** cuando la corrección lo justifique.
7. **Documentación.** Cuatro líneas en el ticket: qué estaba mal, cómo lo encontré, cómo lo corregí, cómo lo verifiqué. Esa es exactamente la rúbrica de hoy.

**Orden por capas.** Se revisa de abajo hacia arriba y se avanza solo cuando la capa actual está sana. Analogía: si el edificio no tiene luz, no se revisa la impresora.

```text
1 Arranque / hardware   ¿la VM enciende, encuentra el disco, llega a GRUB?
2 Kernel                ¿el kernel ve los discos y las interfaces? (dmesg)
3 systemd / servicio    ¿la unidad está activa, habilitada, sin fallos?
4 Red / firewall        ¿la IP, la ruta, el puerto escuchando, el firewall?
5 Permisos              ¿dueño, grupo, rwx, setgid, ACL, umask?
6 SELinux               ¿modo, contexto, puerto, booleano? (AVC)
7 Almacenamiento        ¿espacio, inodos, montaje, fstab, LVM?
8 Aplicación            ¿su configuración, su log propio?
```

**Herramientas por capa.** Esta tabla es la que conviene tener impresa junto al teclado:

| Capa | Pregunta | Primer comando | Segundo comando / detalle |
|---|---|---|---|
| Arranque | ¿Qué falló al arrancar? ¿Cuánto tardó? | `systemctl --failed` | `journalctl -b -p err`, `systemd-analyze time`, `systemd-analyze blame`, `systemd-analyze critical-chain` |
| Kernel / hardware | ¿El kernel ve el disco/la NIC? ¿Hubo OOM? | `sudo dmesg -T \| tail -30` | `sudo dmesg -T \| grep -iE 'error\|fail\|oom\|killed process'`, `lsblk`, `ip link`, `/proc/meminfo` |
| systemd / servicio | ¿Está activo y habilitado? ¿Por qué se cayó? | `systemctl status -l SERVICIO` | `journalctl -u SERVICIO --since "1 hour ago"`, `journalctl -xe`, `systemctl cat SERVICIO`, `systemctl list-dependencies` |
| Red / firewall | ¿Escucha? ¿Se llega? ¿El firewall deja pasar? | `sudo ss -tulpn` | `ip -br a`, `ip route`, `nmcli device`, `nmcli con show`, `sudo firewall-cmd --list-all`, `--get-active-zones`, `sudo lsof -i :80`, `curl -I` |
| Permisos | ¿Quién es el usuario y qué puede tocar? | `ls -ld RUTA` / `id USUARIO` | `namei -l /ruta/completa`, `getfacl`, `passwd -S`, `chage -l`, `sudo -l -U usuario`, `/var/log/secure` |
| SELinux | ¿Está bloqueando? | `sudo ausearch -m AVC -ts recent` | `getenforce`, `ls -Z`, `matchpathcon`, `semanage port -l`, `getsebool -a \| grep X`, `sealert -a /var/log/audit/audit.log` |
| Almacenamiento | ¿Hay espacio? ¿Inodos? ¿Está montado? | `df -h` / `df -i` | `du -xsh /ruta/* \| sort -h`, `sudo lsof +L1` (borrados que ocupan), `findmnt --verify`, `lsblk -f`, `sudo lvs`, `mount -a` |
| Rendimiento | ¿CPU, memoria, disco, carga? | `top` / `uptime` | `vmstat 1 5`, `iostat -x 1 3`, `sar -u`, `free -m`, `ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu \| head` |
| Usuarios / sesiones | ¿Quién entró, quién está, quién falló? | `who` / `w` | `last -n 20`, `sudo lastb`, `journalctl _COMM=sshd`, `sudo faillock` |
| Aplicación | ¿Qué dice ella misma? | su log en `/var/log/APP/` | `apachectl configtest`, `sshd -t`, `rsyslogd -N1`, `strace -p PID` (qué está haciendo un proceso colgado) |

**Tres reglas de oro para hoy:**
- **Reproducir antes de arreglar.** Si no se puede ver el error, no se puede saber si se corrigió.
- **Un cambio a la vez, y anotarlo.** Si se tocan tres cosas y funciona, no se sabe cuál era.
- **`setenforce 0`, `systemctl stop firewalld` y `chmod 777` no son diagnósticos.** Son formas de esconder la causa. En el examen, descalifican.

### Lab 1.1 — Radiografía del servidor en 12 comandos (18 min)

- **Objetivo:** recorrer una vez, en un servidor sano, todos los comandos de la tabla, para reconocer la salida normal y poder distinguir la anormal. Incluye el caso clásico de "el disco está lleno pero `du` no lo explica".

1. Estado general de systemd y del arranque:
```bash
systemctl --failed
systemctl list-units --state=failed --no-legend | wc -l
systemd-analyze time
systemd-analyze blame | head -5
systemd-analyze critical-chain | head -12
```
Salida esperada (aprox.):
```
  UNIT LOAD ACTIVE SUB DESCRIPTION
0 loaded units listed.
0
Startup finished in 1.9s (kernel) + 3.1s (initrd) + 14.7s (userspace) = 19.8s
multi-user.target reached after 14.6s in userspace
4.812s NetworkManager-wait-online.service
1.203s dracut-initqueue.service
  978ms podman-restart.service
  655ms firewalld.service
  412ms sssd.service
multi-user.target @14.6s
└─httpd.service @13.9s +690ms
  └─network-online.target @13.8s
    └─NetworkManager-wait-online.service @9.0s +4.812s
```
Qué observar: `blame` ordena por tiempo propio; `critical-chain` muestra quién esperó a quién. `NetworkManager-wait-online` casi siempre encabeza la lista y es normal. Un `failed` distinto de cero es el primer hilo del que tirar.

2. Errores del arranque actual y del kernel:
```bash
journalctl -p err -b --no-pager | tail -5
sudo dmesg -T | tail -5
sudo dmesg -T | grep -iE 'oom|killed process|error|fail' | tail -5
grep -E 'MemTotal|MemAvailable|SwapFree' /proc/meminfo
```
Salida esperada (aprox.):
```
sep 04 08:01:12 rhel01 kernel: ... (puede estar vacío o mostrar 1-2 líneas inofensivas)
[jue sep  4 08:01:40 2026] IPv6: ADDRCONF(NETDEV_CHANGE): enp0s8: link becomes ready
MemTotal:        3663220 kB
MemAvailable:    3071456 kB
SwapFree:        2097148 kB
```
Qué observar: `-p err` equivale a `-p 0..3` (emerg, alert, crit, err). `dmesg -T` traduce los segundos desde el arranque a fecha legible. Cuando el kernel se queda sin memoria, el **OOM killer** mata el proceso que más consume y deja `Out of memory: Killed process 1234 (java)`: si "un servicio se muere solo a las 3 a.m.", `dmesg` y `journalctl -k` son el primer lugar donde mirar.

3. Sesiones y actividad reciente ("¿qué cambió?"):
```bash
who; w | head -4
last -n 5
sudo lastb | head -5
sudo dnf history | head -5
sudo grep -c 'COMMAND' /var/log/secure
```
Salida esperada (aprox.):
```
student  pts/0        2026-09-04 08:05 (10.0.2.2)
 08:10:33 up 9 min,  1 user,  load average: 0.08, 0.12, 0.09
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
student  pts/0    10.0.2.2         08:05    0.00s  0.09s  0.02s w
student  pts/0        10.0.2.2         Thu Sep  4 08:05   still logged in
reboot   system boot  5.14.0-503.14.1. Thu Sep  4 08:01   still running
btmp begins Thu Sep  4 08:00:01 2026
ID     | Command line              | Date and time    | Action(s)      | Altered
    23 | install -y sysstat        | 2026-09-04 08:07 | Install        |    2
17
```
Qué observar: `last` lee `/var/log/wtmp` (incluye los reinicios: `reboot system boot`), `lastb` lee `/var/log/btmp` (intentos fallidos). `dnf history` responde "¿alguien instaló o actualizó algo antes de que dejara de funcionar?" y `dnf history undo ID` lo revierte.

4. Red y puertos en un vistazo:
```bash
ip -br a
ip route | head -3
sudo ss -tulpn | grep -E ':(22|80|2049|8085) '
sudo firewall-cmd --get-active-zones
sudo lsof -i :80 | head -3
```
Salida esperada (aprox.):
```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 fe80::.../64
enp0s8           UP             192.168.56.10/24 fe80::.../64
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
tcp   LISTEN 0  128   0.0.0.0:22    0.0.0.0:*  users:(("sshd",pid=912,fd=3))
tcp   LISTEN 0  511   *:80          *:*        users:(("httpd",pid=1050,fd=4),...)
tcp   LISTEN 0  64    0.0.0.0:2049  0.0.0.0:*
tcp   LISTEN 0  4096  *:8085        *:*        users:(("rootlessport",pid=1310,fd=10))
internal
  sources: 192.168.56.0/24
public
  interfaces: enp0s3 enp0s8
COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   1050   root    4u  IPv6  25631      0t0  TCP *:http (LISTEN)
httpd   1051 apache    4u  IPv6  25631      0t0  TCP *:http (LISTEN)
```
Qué observar: en UTM las interfaces son `enp0s1`/`enp0s2` (verificar con `nmcli device`). `ss -tulpn` responde tres preguntas de golpe: ¿escucha?, ¿en qué IP?, ¿qué proceso? Si el puerto **no** aparece aquí, el firewall no es el problema todavía.

5. Espacio: el caso del archivo borrado que sigue ocupando. Primero, provocarlo. Se usa `/datos` (LV `lv_datos` del Día 6); quien no lo tenga montado trabaja sobre `/tmp` y sustituye `/datos` por `/tmp` en todo el paso:
```bash
findmnt /datos || echo "SIN /datos: usar /tmp en este paso"
df -h /datos | tail -1
sudo fallocate -l 300M /datos/temporal.img
df -h /datos | tail -1
sudo tail -f /datos/temporal.img > /dev/null &
sudo rm -f /datos/temporal.img
df -h /datos | tail -1
sudo du -sh /datos
```
Salida esperada (aprox.):
```
TARGET SOURCE                        FSTYPE OPTIONS
/datos /dev/mapper/vg_datos-lv_datos xfs    rw,relatime,attr2,inode64,logbufs=8,...
/dev/mapper/vg_datos-lv_datos  2.0G   47M  2.0G   3% /datos
/dev/mapper/vg_datos-lv_datos  2.0G  347M  1.7G  18% /datos
[1] 4821
/dev/mapper/vg_datos-lv_datos  2.0G  347M  1.7G  18% /datos
47M	/datos
```
Qué observar: `df` dice 347M usados, `du` dice 47M. **La diferencia es un archivo borrado que un proceso mantiene abierto**: el espacio no se libera hasta que ese proceso cierre el descriptor. Es la causa número uno de "borré el log gigante y el disco sigue lleno".

6. Encontrarlo y liberarlo sin matar el proceso (o matándolo):
```bash
sudo lsof +L1 | grep -v '^COMMAND' | head -3
PID=$(sudo lsof +L1 -t | head -1); echo "PID=$PID"
sudo ls -l /proc/$PID/fd | grep deleted
FD=$(sudo ls -l /proc/$PID/fd | awk '/deleted/{print $9; exit}')
sudo sh -c ": > /proc/$PID/fd/$FD"
df -h /datos | tail -1
sudo kill $PID; sleep 1; jobs
```
Salida esperada (aprox.):
```
tail    4821 root    3r   REG  253,2 314572800     0    69 /datos/temporal.img (deleted)
PID=4821
lr-x------. 1 root root 64 Sep  4 08:14 3 -> /datos/temporal.img (deleted)
/dev/mapper/vg_datos-lv_datos  2.0G   47M  2.0G   3% /datos
[1]+  Terminado               sudo tail -f /datos/temporal.img > /dev/null
```
(Según la shell, la última línea puede aparecer como `Terminated`, `Hecho` o `Done`: lo importante es que el trabajo `[1]` ya no está.)
Qué observar: `lsof +L1` lista archivos abiertos con **menos de un enlace** (borrados). Truncar por `/proc/PID/fd/N` libera el espacio sin reiniciar el servicio; reiniciar el servicio (o `kill`) también lo libera. Regla: los logs no se borran con `rm`, se vacían con `: > archivo` o se rotan con logrotate.

7. Rendimiento con `sysstat` (instalar; `sar` acumula datos cada 10 min a partir de ahora):
```bash
sudo dnf install -y sysstat strace lsof
sudo systemctl enable --now sysstat
sudo systemctl enable --now sysstat-collect.timer sysstat-summary.timer
systemctl list-timers 'sysstat*' --no-pager | head -3
vmstat 1 3
iostat -x 1 2 | grep -A2 Device | tail -3
sar -u 1 3 | tail -2
top -b -n 1 | head -12
```
Salida esperada (aprox.):
```
NEXT                        LEFT    LAST  PASSED  UNIT                    ACTIVATES
Thu 2026-09-04 08:20:00 EST 3min    n/a   n/a     sysstat-collect.timer   sysstat-collect.service
Fri 2026-09-05 00:07:00 EST 15h     n/a   n/a     sysstat-summary.timer   sysstat-summary.service
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 2951204   2588 424556    0    0   112    18  145  260  1  1 98  0  0
Device            r/s     rkB/s   ...  %util
sda              1.20     45.10   ...   0.30
Average:        all      0.67      0.00      0.50      0.00      0.00     98.83
top - 08:16:02 up 15 min,  1 user,  load average: 0.05, 0.10, 0.09
Tasks: 138 total,   1 running, 137 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.3 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem :   3577.4 total,   2881.1 free,    311.7 used,    384.6 buff/cache
```
Qué observar: en `vmstat`, `r` (procesos esperando CPU) mayor que `nproc` sostenido = falta CPU; `si/so` distintos de cero = está usando swap; `wa` alto = espera de disco. `iostat -x` con `%util` cerca de 100 = disco saturado. El servicio `sysstat` solo prepara el archivo del día; **quien recoge los datos cada 10 minutos es `sysstat-collect.timer`** (por eso se habilita también). `sar -u` sin argumentos leerá `/var/log/sa/saDD` mañana, cuando ya tenga histórico ("¿a qué hora se disparó la CPU anoche?"); hoy, recién instalado, respondería `Cannot open /var/log/sa/saNN: No such file or directory`, y por eso se le pasan intervalos (`sar -u 1 3`). En UTM el disco se llama `vda`.

8. Mención de `strace` (no profundizar): cuando un proceso "no hace nada" o "se cuelga", `sudo strace -p PID -f -e trace=network,file` muestra en qué llamada al sistema está detenido (por ejemplo, esperando un `connect()` a una base de datos que no responde). Probarlo 5 segundos sobre `sshd`:
```bash
sudo timeout 5 strace -p $(pgrep -o sshd) -e trace=network 2>&1 | head -5
```
Salida esperada: unas líneas `accept4(...)` o `select(...)` y `strace: Process ... detached`. Qué observar: `sshd` está en `accept4`: esperando conexiones; es lo normal.

- **Checkpoint:** pegar en el chat la salida de:
```bash
systemctl --failed --no-legend | wc -l; df -h /datos | tail -1; systemctl is-active sysstat; rpm -q sysstat strace lsof | wc -l
```
Esperado: `0`, la línea de `/datos` con ~3 % de uso, `active`, `3`.

---

## Bloque 2 — Logs avanzados (40 min)

### Conceptos (6 min)

**Dos sistemas de log conviven en RHEL 9, y se alimentan uno al otro.**
- **journald** (`systemd-journald`) recoge **todo**: stdout/stderr de cada servicio, mensajes del kernel, syslog, auditoría. Guarda en binario, indexado por campos (`_SYSTEMD_UNIT`, `_PID`, `_UID`, `_COMM`, `PRIORITY`, `SYSLOG_IDENTIFIER`), y por eso `journalctl` filtra tan rápido. En RHEL 9 el journal es **persistente por defecto** porque existe `/var/log/journal/` (`Storage=auto` en `/etc/systemd/journald.conf`); si ese directorio no existe, vive en `/run/log/journal` y se pierde al reiniciar.
- **rsyslog** lee del journal (módulo `imjournal`) y escribe los archivos de texto de `/var/log/` (`messages`, `secure`, `cron`, `maillog`, `boot.log`) según reglas **facilidad.prioridad → destino**. También **envía** a un servidor central (`@` UDP, `@@` TCP) y **recibe** de otros (`imudp`, `imtcp`). Es lo que un SIEM o un colector institucional espera.
- **auditd** es aparte: registra llamadas al sistema y accesos a archivos según reglas del kernel (y las denegaciones de SELinux, los AVC). Escribe `/var/log/audit/audit.log`; se consulta con `ausearch`/`aureport`, no con `grep` (los timestamps son epoch: `ausearch -i` los traduce).
- **logrotate** evita que `/var/log` llene el disco: rota, comprime y borra por edad o tamaño. En RHEL 9 lo dispara `logrotate.timer` a diario.

Analogía: journald es la **grabadora** de todo el edificio; rsyslog es el **archivista** que copia lo importante a carpetas y lo manda a la central; auditd es la **cámara de seguridad** que solo graba las puertas que se le indican; logrotate es el que **vacía el archivero** cada noche.

**Facilidades y prioridades (syslog).** Facilidad = quién habla (`auth`, `authpriv`, `cron`, `kern`, `mail`, `daemon`, `user`, `local0`–`local7`). Prioridad = qué tan grave (`debug < info < notice < warning < err < crit < alert < emerg`). Una regla `mail.warning /var/log/mail-avisos` captura `warning` **y superiores**; `mail.=warning` solo esa; `mail.none` excluye. `*.info;mail.none;authpriv.none;cron.none /var/log/messages` es la regla real de `messages`: por eso el correo, la autenticación y cron van a sus propios archivos.

### Lab 2.1 — journald a fondo (9 min)

- **Objetivo:** navegar por arranques anteriores, filtrar por campos, ver el registro completo de un mensaje, controlar el tamaño del journal y confirmar la persistencia.

1. Arranques disponibles y errores del arranque anterior:
```bash
journalctl --list-boots | tail -3
journalctl -b -1 -p err --no-pager | tail -3
journalctl -b -1 -n 3 --no-pager
```
Salida esperada (aprox.):
```
-2 6a2c...  Wed 2026-09-03 18:02:10 EST—Wed 2026-09-03 21:40:55 EST
-1 9f10...  Thu 2026-09-04 07:30:01 EST—Thu 2026-09-04 07:58:30 EST
 0 c4d7...  Thu 2026-09-04 08:01:02 EST—Thu 2026-09-04 08:20:11 EST
sep 03 21:40:50 rhel01 systemd[1]: ... (últimas líneas del arranque anterior: cómo se apagó)
```
Qué observar: `-b 0` (o `-b`) es el arranque actual, `-b -1` el anterior. Las **últimas** líneas de `-b -1` dicen si el servidor se apagó bien (`Reached target Power-Off`) o se cortó (nada: se fue la luz o se colgó). Si `--list-boots` solo muestra `0`, el journal no es persistente.

2. Filtros por campo (los "índices" del journal). Ejecutar y comparar con `-u`:
```bash
journalctl -u httpd -b --no-pager | tail -2
journalctl _SYSTEMD_UNIT=httpd.service -b --no-pager | tail -2
journalctl _COMM=sudo -n 3 --no-pager
journalctl _UID=1000 -n 2 --no-pager
journalctl _PID=1 -b --no-pager | head -3
journalctl -k -n 2 --no-pager
journalctl -t backup --since yesterday --no-pager | tail -2
```
Salida esperada (aprox.):
```
sep 04 08:01:14 rhel01 systemd[1]: Started The Apache HTTP Server.
sep 04 08:01:14 rhel01 httpd[1050]: Server configured, listening on: port 80
sep 04 08:07:20 rhel01 sudo[3120]:  student : TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/dnf install -y sysstat
sep 04 08:07:20 rhel01 sudo[3120]: pam_unix(sudo:session): session opened for user root(uid=0) by student(uid=1000)
sep 04 08:01:05 rhel01 systemd[1]: Starting Journal Service...
sep 04 08:01:02 rhel01 kernel: Linux version 5.14.0-503.14.1.el9_5.x86_64 ...
sep 03 23:30:01 rhel01 backup[9102]: OK: creado /backups/empresa-2026-09-03-2330.tar.gz (4.0K)
```
Qué observar: `-u X` es un atajo de `_SYSTEMD_UNIT=X.service` que además incluye los mensajes que systemd escribe **sobre** la unidad (por eso `-u` suele mostrar más). `_COMM` es el nombre del ejecutable, `-t` el identificador syslog (lo que `logger -t` pone). `-k` = `_TRANSPORT=kernel` = `dmesg` con fechas y de arranques anteriores (`-k -b -1`: los mensajes del kernel del arranque anterior, algo que `dmesg` no puede dar).

3. Ver todos los campos de un mensaje y exportar en JSON (para un SIEM o un script):
```bash
journalctl -t backup -n 1 -o verbose --no-pager | head -20
journalctl -t backup -n 1 -o json-pretty --no-pager | grep -E '"(MESSAGE|_COMM|_UID|_HOSTNAME|PRIORITY)"'
journalctl -p warning -b -o short-iso --no-pager | tail -2
```
Salida esperada (aprox.):
```
Thu 2026-09-03 23:30:01.412345 EST [s=...;i=...;b=...;m=...;t=...;x=...]
    PRIORITY=5
    SYSLOG_FACILITY=1
    SYSLOG_IDENTIFIER=backup
    _UID=1000
    _GID=1000
    _COMM=logger
    _EXE=/usr/bin/logger
    _HOSTNAME=rhel01
    MESSAGE=OK: creado /backups/empresa-2026-09-03-2330.tar.gz (4.0K)
    ...
        "MESSAGE" : "OK: creado /backups/empresa-2026-09-03-2330.tar.gz (4.0K)",
        "PRIORITY" : "5",
        "_COMM" : "logger",
        "_HOSTNAME" : "rhel01",
        "_UID" : "1000",
2026-09-04T08:01:20-0500 rhel01 kernel: ... (algún warning del arranque)
```
Qué observar: `-o verbose` responde "¿qué campos puedo usar para filtrar?": cualquiera de esas líneas `CAMPO=valor` sirve tal cual en la línea de comandos (`journalctl _EXE=/usr/bin/logger`). `PRIORITY=5` y `SYSLOG_FACILITY=1` son los valores por defecto de `logger` (`user.notice`); con `logger -p local5.err` serían `PRIORITY=3` y `SYSLOG_FACILITY=21`. `journalctl -F _COMM` lista todos los valores que ha visto un campo (útil para descubrir nombres).

Si `journalctl -t backup` no devuelve nada (la VM estuvo apagada a las 23:30 y el cron del Día 7 no llegó a correr), generar el mensaje ahora para tener todos la misma salida:
```bash
logger -t backup "OK: creado /backups/empresa-$(date +%F-%H%M).tar.gz (4.0K)"
journalctl -t backup -n 1 --no-pager
```

4. Tamaño, retención y configuración:
```bash
journalctl --disk-usage
ls -ld /var/log/journal/*/ | head -2
sudo journalctl --vacuum-time=2weeks
grep -E '^#?(Storage|SystemMaxUse|MaxRetentionSec|Compress)=' /etc/systemd/journald.conf
```
Salida esperada (aprox.):
```
Archived and active journals take up 96.0M in the file system.
drwxr-sr-x. 2 root systemd-journal 4096 Sep  4 08:01 /var/log/journal/3a7c.../
Vacuuming done, freed 0B of archived journals from /var/log/journal/3a7c...
#Storage=auto
#Compress=yes
#SystemMaxUse=
#MaxRetentionSec=
```
Qué observar: `--vacuum-time` (y `--vacuum-size=500M`) solo borra archivos **archivados**, nunca el activo. Los valores comentados son los defaults: `Storage=auto` = persistente si existe `/var/log/journal`; `SystemMaxUse` por defecto el 10 % del filesystem (máximo 4 GB). Para limitar: crear un *drop-in* en vez de editar el archivo (es la costumbre en RHEL 9 y sobrevive a actualizaciones):
```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/pgn.conf > /dev/null <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=500M
MaxRetentionSec=1month
EOF
sudo systemctl restart systemd-journald
journalctl --disk-usage
```
Salida esperada: `Archived and active journals take up ...` sin errores. Qué observar: si el journal **no** fuera persistente, la receta del examen es `mkdir -p /var/log/journal` + `systemctl restart systemd-journald` (o `journalctl --flush`); `Storage=persistent` obliga a crearlo aunque no exista el directorio.

- **Checkpoint:** pegar en el chat la salida de:
```bash
journalctl --list-boots | wc -l; journalctl --disk-usage; grep SystemMaxUse /etc/systemd/journald.conf.d/pgn.conf
```

### Lab 2.2 — rsyslog: reglas, plantillas y log remoto con la misma VM (10 min)

- **Objetivo:** escribir una regla propia, entender la estructura de `/etc/rsyslog.conf`, y montar la VM como **servidor** de logs (TCP 514) y **cliente** de sí misma, con el detalle que evita el bucle infinito.

1. Anatomía de la configuración:
```bash
grep -nE '^module|^input|^\$IncludeConfig|^include' /etc/rsyslog.conf
grep -nE '^(\*\.|authpriv|mail|cron|local7)' /etc/rsyslog.conf
ls /etc/rsyslog.d/
sudo rsyslogd -N1 2>&1 | tail -1
```
Salida esperada (aprox.):
```
17:module(load="imuxsock"     # provides support for local system logging (e.g. via logger command)
19:module(load="imjournal"    # provides access to the systemd journal
40:include(file="/etc/rsyslog.d/*.conf" mode="optional")
48:*.info;mail.none;authpriv.none;cron.none                /var/log/messages
51:authpriv.*                                              /var/log/secure
54:mail.*                                                  -/var/log/maillog
57:cron.*                                                  /var/log/cron
60:*.emerg                                                 :omusrmsg:*
66:local7.*                                                /var/log/boot.log
21-cloudinit.conf  (o vacío)
rsyslogd: End of config validation run. Bye.
```
Qué observar: rsyslog 8 mezcla dos sintaxis: **RainerScript** (`module(...)`, `input(...)`, `action(...)`) y la **clásica** (`facilidad.prioridad  destino`). Ambas son válidas y conviven. El guion en `-/var/log/maillog` significa "sin sync por cada línea" (más rápido). `rsyslogd -N1` es el `configtest` de rsyslog: **siempre** antes de reiniciar.

2. Regla propia con facilidad `local5` para la aplicación institucional (la del Día 4 usó `local0`; hoy otra para no pisarla), más una plantilla simple de formato. **El orden importa**: los archivos de `/etc/rsyslog.d/` se incluyen **antes** de las reglas clásicas de `/etc/rsyslog.conf` (la línea `include(...)` está en la sección GLOBAL DIRECTIVES, en el ~40, y las reglas empiezan en el ~48), así que lo que aquí no se detenga con `stop` seguirá su camino hacia `/var/log/messages`:
```bash
sudo tee /etc/rsyslog.d/pgn-app.conf > /dev/null <<'EOF'
# Aplicacion institucional (facilidad local5).
# Plantilla simple: fecha ISO, host, programa, mensaje
template(name="FormatoPGN" type="string"
         string="%TIMESTAMP:::date-rfc3339% %HOSTNAME% %programname%[%procid%]: %msg%\n")

# 1) Solo los errores se copian ademas a messages (regla clasica, formato tradicional)
local5.err  /var/log/messages

# 2) Todo local5 va a su propio archivo con nuestra plantilla
local5.*    action(type="omfile" file="/var/log/pgn-app.log" template="FormatoPGN")

# 3) Y aqui termina: no siga hasta las reglas de /etc/rsyslog.conf (evita duplicados)
local5.*    stop
EOF
sudo rsyslogd -N1 2>&1 | tail -1
sudo systemctl restart rsyslog
logger -p local5.info -t portal-pgn "Usuario ana consulto expediente 2026-001"
logger -p local5.err  -t portal-pgn "Fallo de conexion a la base de datos"
sudo tail -2 /var/log/pgn-app.log
sudo grep portal-pgn /var/log/messages | tail -1
```
Salida esperada (aprox.):
```
rsyslogd: End of config validation run. Bye.
2026-09-04T08:31:10.12345-05:00 rhel01 portal-pgn[5210]: Usuario ana consulto expediente 2026-001
2026-09-04T08:31:11.98765-05:00 rhel01 portal-pgn[5213]: Fallo de conexion a la base de datos
Sep  4 08:31:11 rhel01 portal-pgn[5213]: Fallo de conexion a la base de datos
```
Qué observar: `local5.* stop` es la acción "deja de evaluar este mensaje": sin ella, `local5.info` habría caído además en la regla `*.info` de `/etc/rsyslog.conf` y aparecería **duplicado** en `messages`. Por eso `pgn-app.log` tiene las dos líneas y `messages` solo la de `err`. Las propiedades `%TIMESTAMP%`, `%HOSTNAME%`, `%programname%`, `%procid%`, `%msg%` son los campos que rsyslog conoce de cada mensaje; `man rsyslog.conf` y `/usr/share/doc/rsyslog/` tienen la lista completa. Ejercicio de 30 segundos: comentar la línea `local5.* stop`, `sudo systemctl restart rsyslog`, repetir el `logger -p local5.info` y ver cómo ahora sí aparece en `messages`.

3. **Servidor** de logs: recibir por TCP 514. El truco es atar la entrada a un **ruleset propio** para que lo recibido solo se escriba en un archivo y no vuelva a pasar por las reglas normales (que, en el paso 4, reenviarán todo a este mismo servidor: sin el ruleset, la VM se enviaría sus propios mensajes a sí misma para siempre):
```bash
sudo tee /etc/rsyslog.d/remoto.conf > /dev/null <<'EOF'
# Servidor central de logs (colector). Recibe por TCP 514.
module(load="imtcp")
ruleset(name="desde_remotos") {
    action(type="omfile" file="/var/log/remoto.log")
}
input(type="imtcp" port="514" ruleset="desde_remotos")
EOF
sudo rsyslogd -N1 2>&1 | tail -1
sudo semanage port -l | grep -E 'syslogd_port_t|rsh_port_t'
sudo semanage port -m -t syslogd_port_t -p tcp 514
sudo semanage port -l | grep syslogd_port_t
sudo systemctl restart rsyslog
sudo ss -tlnp | grep ':514 '
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=514/tcp
sudo firewall-cmd --reload
```
Salida esperada (aprox.):
```
rsyslogd: End of config validation run. Bye.
rsh_port_t                     tcp      514
syslogd_port_t                 tcp      601, 20514
syslogd_port_t                 udp      514, 601, 20514
syslogd_port_t                 tcp      514, 601, 20514
syslogd_port_t                 udp      514, 601, 20514
tcp   LISTEN 0  25   0.0.0.0:514   0.0.0.0:*   users:(("rsyslogd",pid=5301,fd=5))
success
success
success
```
Qué observar: **514/tcp no es de syslog para SELinux**: es `rsh_port_t`, herencia histórica del `rsh` (514/udp sí es `syslogd_port_t`). Por eso se hace `semanage port -m` (modificar, no `-a` añadir: el puerto ya está definido y `-a` daría `ValueError: Port tcp/514 already defined`). Se deshace con `sudo semanage port -d -p tcp 514`, que lo devuelve a su valor de fábrica. ⚠️ **Verificar en la VM antes de la clase**: en algunas builds de la política rsyslog logra enlazar 514/tcp sin este cambio; hacerlo igual no rompe nada y es el procedimiento correcto. Si `ss` **no** muestra el puerto, la prueba es `sudo ausearch -m AVC -ts recent | grep rsyslogd` buscando `name_bind`, y `sudo journalctl -u rsyslog -n 5` mostrará `could not create listen socket ... Permission denied`.

Sobre el firewall: se abre en **las dos zonas** porque el NAT entra por `public` y la red host-only por `internal` (regla establecida el Día 8). Para el laboratorio de un solo equipo el firewall ni siquiera interviene —el tráfico a la IP propia sale y entra por `lo`, que está en la zona `trusted`—; se abre para la demo del instructor entre dos VMs. Para syslog por **UDP** existe el servicio predefinido `syslog` (`firewall-cmd --add-service=syslog`); para TCP hay que abrir el puerto.

4. **Cliente**: reenviar todo al colector. Usar la IP host-only propia (o `127.0.0.1` si no hay host-only); en clase, el instructor puede dar la IP de su VM para que todos envíen a su colector (ver Notas):
```bash
sudo tee /etc/rsyslog.d/cliente.conf > /dev/null <<'EOF'
# Cliente: enviar todo al colector por TCP (@@). Con @ seria UDP.
*.*  @@192.168.56.10:514
EOF
sudo rsyslogd -N1 2>&1 | tail -1
sudo systemctl restart rsyslog
logger -t prueba-local "Mensaje generado localmente, viaja por rsyslog al colector"
logger -n 192.168.56.10 -P 514 -T -t prueba-directa "Mensaje enviado directo por TCP con logger -n"
sleep 1; sudo tail -3 /var/log/remoto.log
sudo wc -l /var/log/remoto.log
```
Salida esperada (aprox.):
```
rsyslogd: End of config validation run. Bye.
Sep  4 08:35:02 rhel01 prueba-local[5410]: Mensaje generado localmente, viaja por rsyslog al colector
Sep  4 08:35:03 rhel01 prueba-directa Mensaje enviado directo por TCP con logger -n
Sep  4 08:35:03 rhel01 systemd[1]: ... (mensajes del sistema que también llegan)
42 /var/log/remoto.log
```
Qué observar: los dos caminos funcionan: `prueba-local` fue journal → rsyslog local → regla `*.*` → TCP → ruleset `desde_remotos` → `remoto.log`; `prueba-directa` salió de `logger -n` directo al puerto 514 (`logger` con `-n` envía en formato RFC 5424, con fecha ISO, pero el colector lo **vuelve a escribir** con su plantilla por defecto `RSYSLOG_TraditionalFileFormat`, por eso en el archivo las dos líneas se ven con el formato clásico `Sep  4 08:35:03`). `remoto.log` crece con todo el sistema: es lo que se vería en el colector institucional con decenas de servidores; ahí es donde una plantilla por host (`/var/log/remoto/%HOSTNAME%/%programname%.log`, con `dynaFile`) ordena la carpeta. Si el archivo no crece: `sudo ss -tnp | grep 514` (¿hay conexión establecida?), `sudo journalctl -u rsyslog -n 5` (¿`omfwd` reporta `connection refused`?), firewall.

5. Dejar el cliente apagado para no duplicar todo el log durante el resto del día (el servidor puede quedarse):
```bash
sudo mv /etc/rsyslog.d/cliente.conf /root/cliente.conf.off
sudo systemctl restart rsyslog; systemctl is-active rsyslog
```

- **Checkpoint:** pegar en el chat la salida de:
```bash
sudo grep -c prueba /var/log/remoto.log; sudo ss -tlnp | grep -c ':514 '; sudo tail -1 /var/log/pgn-app.log
```
Esperado: `2` (o más), `1`, la línea de "Fallo de conexion".

### Lab 2.3 — logrotate a fondo (7 min)

- **Objetivo:** rotar el log de `monitor-disco.sh` (Día 7) a diario con 7 copias comprimidas, probar en seco con `-d`, forzar con `-f` y entender el archivo de estado.

1. Asegurar que el log existe y tiene contenido (quien no haya hecho el Día 7 lo genera igual; todos tendrán las mismas 40 líneas):
```bash
sudo touch /var/log/monitor-disco.log
sudo truncate -s 0 /var/log/monitor-disco.log   # para que todos tengan exactamente 40 lineas
for i in $(seq 40 -1 1); do
  echo "$(date -d "-$i hour" '+%F %T') $([ $((i % 7)) -eq 0 ] && echo 'ALERTA: /backups al 9'$((i % 10))'% (umbral 80%)' || echo 'OK: / al 1'$((i % 10))'%')" | sudo tee -a /var/log/monitor-disco.log > /dev/null
done
sudo wc -l /var/log/monitor-disco.log; sudo tail -2 /var/log/monitor-disco.log
cat /etc/logrotate.conf | grep -vE '^#|^$' | head -8
ls /etc/logrotate.d/ | head; systemctl list-timers logrotate.timer --no-pager | head -2
```
Salida esperada (aprox.):
```
40 /var/log/monitor-disco.log
2026-09-04 07:36:00 OK: / al 12%
2026-09-04 08:36:00 OK: / al 11%
weekly
rotate 4
create
dateext
include /etc/logrotate.d
btmp  chrony  dnf  firewalld  httpd  psacct  rsyslog  samba  sssd  wtmp
NEXT                        LEFT     LAST                        PASSED  UNIT            ACTIVATES
Fri 2026-09-05 00:00:00 EST 15h left Thu 2026-09-04 00:00:00 EST 8h ago  logrotate.timer logrotate.service
```
Qué observar: `logrotate.conf` fija los defaults (`weekly`, `rotate 4`, `dateext` = sufijo con fecha en vez de `.1`); cada archivo de `/etc/logrotate.d/` los sobreescribe para sus logs. El timer corre a medianoche; `logrotate` decide qué toca según `/var/lib/logrotate/logrotate.status`.

2. Escribir la política:
```bash
sudo tee /etc/logrotate.d/monitor > /dev/null <<'EOF'
/var/log/monitor-disco.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    postrotate
        /usr/bin/logger -t logrotate "monitor-disco.log rotado"
    endscript
}
EOF
sudo logrotate -d /etc/logrotate.d/monitor 2>&1 | grep -E 'rotating|log does not need|considering'
```
Salida esperada (aprox.):
```
rotating pattern: /var/log/monitor-disco.log  after 1 days (7 rotations)
considering log /var/log/monitor-disco.log
  log does not need rotating (log has already been rotated)   <- o "Now: ... Last rotated at ..."
```
Qué observar: `-d` (debug) **no hace nada**: solo explica qué haría y por qué. Es la manera segura de validar sintaxis y decisión. Ojo con un detalle que confunde: al invocar `logrotate` con **solo este fragmento** no se leen los valores globales de `/etc/logrotate.conf` (`weekly`, `rotate 4`, `dateext`); vale únicamente lo escrito dentro de las llaves. Cuando lo ejecuta `logrotate.timer` sí se parte de `/etc/logrotate.conf`, que hace `include /etc/logrotate.d`, y entonces `dateext` está activo. `delaycompress` deja la última rotación sin comprimir (el programa puede estar aún escribiendo); `create 0640 root root` crea el archivo nuevo con esos permisos; `missingok` evita error si el log no existe; `notifempty` no rota vacíos. Otras opciones útiles: `size 10M`, `maxage 30`, `copytruncate` (para programas que no reabren el archivo; en su lugar copia y vacía).

3. Forzar dos rotaciones para ver la secuencia completa:
```bash
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor-disco.log*
echo "$(date '+%F %T') OK: / al 11% (linea nueva tras rotar)" | sudo tee -a /var/log/monitor-disco.log > /dev/null
sudo logrotate -f /etc/logrotate.d/monitor
ls -l /var/log/monitor-disco.log*
grep monitor-disco /var/lib/logrotate/logrotate.status
journalctl -t logrotate -n 2 --no-pager
```
Salida esperada (aprox.):
```
-rw-r-----. 1 root root    0 Sep  4 08:40 /var/log/monitor-disco.log
-rw-r--r--. 1 root root 1960 Sep  4 08:36 /var/log/monitor-disco.log.1
-rw-r-----. 1 root root    0 Sep  4 08:41 /var/log/monitor-disco.log
-rw-r-----. 1 root root   58 Sep  4 08:40 /var/log/monitor-disco.log.1
-rw-r--r--. 1 root root  312 Sep  4 08:36 /var/log/monitor-disco.log.2.gz
"/var/log/monitor-disco.log" 2026-9-4-8:41:0
sep 04 08:40:11 rhel01 logrotate[5620]: monitor-disco.log rotado
sep 04 08:41:02 rhel01 logrotate[5644]: monitor-disco.log rotado
```
Qué observar: la primera rotación no comprime (`delaycompress`), solo renombra a `.1`; la segunda desplaza `.1` a `.2` y **entonces** comprime la `.2`. Los sufijos son numéricos porque, como se dijo, invocar el fragmento suelto no aplica el `dateext` global; cuando lo haga el timer, el nombre será `monitor-disco.log-20260905`. Con `dateext` dos rotaciones **el mismo día** chocan de nombre (`error: destination ... already exists`): en producción no ocurre porque solo hay una diaria. `postrotate` es donde va `systemctl kill -s HUP rsyslog` o `systemctl reload httpd` cuando el programa mantiene el archivo abierto.

- **Checkpoint:** pegar en el chat la salida de:
```bash
ls /var/log/monitor-disco.log* | wc -l; sudo logrotate -d /etc/logrotate.d/monitor 2>&1 | grep -c error
```
Esperado: `3` (el activo, `.1` y `.2.gz`), y `0`.

### Lab 2.4 — auditd y `/var/log/secure`: quién tocó qué (8 min)

- **Objetivo:** vigilar `/etc/passwd` y `/etc/shadow` con una regla persistente de auditd, buscar por llave y por usuario, y extraer de `/var/log/secure` los intentos de acceso fallidos por IP.

1. Estado de auditd y regla persistente con llave (`-k`):
```bash
systemctl is-active auditd; sudo auditctl -s | grep -E 'enabled|pid'
sudo tee /etc/audit/rules.d/pgn.rules > /dev/null <<'EOF'
## Reglas PGN: cambios en cuentas y en sudoers
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k passwd_changes
-w /etc/group -p wa -k passwd_changes
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers
EOF
sudo augenrules --load
sudo auditctl -l
```
Salida esperada (aprox.):
```
active
enabled 1
pid 690
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k passwd_changes
-w /etc/group -p wa -k passwd_changes
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d -p wa -k sudoers
```
(`augenrules --load` no imprime nada cuando carga bien; si ya estaban cargadas y el archivo no cambió, responde `/sbin/augenrules: No change`. Las cinco líneas son la salida de `auditctl -l`.)
Qué observar: `-w ruta -p wa` = vigilar escritura (`w`) y cambio de atributos (`a`); `-k` etiqueta los eventos para buscarlos después. `augenrules --load` compila `/etc/audit/rules.d/*.rules` en `/etc/audit/audit.rules` y los carga; los muestra `auditctl -l`. Las reglas sobreviven al reinicio porque `auditd.service` ejecuta `augenrules --load` al arrancar. Si `auditctl -s` mostrara `enabled 2`, las reglas son inmutables hasta reiniciar (`-e 2` al final del archivo: se usa en servidores endurecidos).

2. Provocar eventos y buscarlos por llave, por usuario y por fecha:
```bash
sudo useradd -c "Prueba auditoria" prueba_audit
echo 'prueba_audit:Pgn2026!lab' | sudo chpasswd
sudo ausearch -k passwd_changes -ts recent -i | grep -E '^type=(SYSCALL|PATH)' | grep -oE 'comm=[^ ]+|name=[^ ]+|auid=[^ ]+|exe=[^ ]+' | sort | uniq -c | sort -rn | head -6
sudo ausearch -ua $(id -u) -ts today -i | grep -c 'type=SYSCALL'
sudo ausearch -m USER_CMD -ts recent -i | grep -oE 'cmd=[^ ]+' | tail -3
sudo aureport --summary | head -14
sudo aureport -au --summary | head -6
```
Salida esperada (aprox.):
```
      4 auid=student
      2 name=/etc/shadow
      2 name=/etc/passwd
      2 exe=/usr/sbin/useradd
      2 comm=useradd
      1 exe=/usr/sbin/chpasswd
57
cmd=useradd -c "Prueba auditoria" prueba_audit
cmd=chpasswd
Summary Report
======================
Range of time in logs: 09/03/2026 18:02:10.114 - 09/04/2026 08:45:30.913
Number of changes in configuration: 12
Number of changes to accounts, groups, or roles: 4
Number of logins: 3
Number of failed logins: 0
Number of authentications: 9
Number of failed authentications: 1
Number of users: 3
Number of terminals: 4
Number of host names: 2
Number of executables: 22
Number of commands: 15
...
Authentication Report
============================================
# date time acct host term exe success event
```
Qué observar: `auid` (audit uid) es **quién inició la sesión**, aunque después hiciera `sudo`: la auditoría no se pierde con `sudo -i`. `-ts recent` = últimos 10 minutos, `-ts today`, `-ts 08:00`. `-i` interpreta uids y syscalls. `-m USER_CMD` son los comandos ejecutados vía `sudo` (auditd los registra aparte de `/var/log/secure`). `aureport` resume: es el informe que se adjunta a un incidente.

3. Intentos de acceso SSH fallidos en `/var/log/secure` (guiño al pentester: esto es lo primero que se mira tras una alerta de fuerza bruta). Como el Día 8 dejó `PasswordAuthentication no`, un intento con usuario inexistente aparece como `Invalid user`; generar tres y contarlos por IP:
```bash
for u in admin oracle test; do ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no $u@localhost true 2>/dev/null; done
sudo grep -E 'Invalid user|Failed|authentication failure' /var/log/secure | tail -4
sudo grep 'Invalid user' /var/log/secure | awk '{print $(NF-2)}' | sort | uniq -c | sort -rn
sudo lastb | head -4
```
Salida esperada (aprox.):
```
Sep  4 08:47:01 rhel01 sshd[5801]: Invalid user admin from 127.0.0.1 port 40212
Sep  4 08:47:01 rhel01 sshd[5801]: Connection closed by invalid user admin 127.0.0.1 port 40212 [preauth]
Sep  4 08:47:02 rhel01 sshd[5804]: Invalid user oracle from 127.0.0.1 port 40214
Sep  4 08:47:02 rhel01 sshd[5807]: Invalid user test from 127.0.0.1 port 40216
      3 127.0.0.1
btmp begins Thu Sep  4 08:00:01 2026   (puede estar vacío: los intentos por clave no pasan por PAM)
```
Qué observar: el `-4` de `ssh` obliga a IPv4; sin él, `localhost` resuelve primero a `::1` y las líneas dirían `from ::1`, que rompe el conteo por IP de abajo. Como el Día 8 dejó `AllowUsers student`, aunque el usuario existiera tampoco entraría: para usuarios inexistentes sshd registra primero `Invalid user`, que es lo que se busca aquí.
Y sobre un `secure` de un servidor con contraseñas habilitadas (formato `Failed password for [invalid user] USUARIO from IP port N ssh2`), que se genera para tener todos el mismo archivo:
```bash
for i in $(seq 1 30); do
  ip=$(shuf -e 45.33.12.9 185.220.101.4 10.0.2.2 200.46.30.77 -n 1)
  u=$(shuf -e root admin student oracle -n 1)
  if [ "$u" = student ]; then quien="$u"; else quien="invalid user $u"; fi
  printf 'Sep  3 %02d:%02d:%02d rhel01 sshd[%d]: Failed password for %s from %s port %d ssh2\n' \
         "$((i % 9))" "$((10 + i % 50))" "$((i % 60))" "$((6000 + i))" "$quien" "$ip" "$((40000 + i))"
done > ~/secure-ejemplo.log
head -3 ~/secure-ejemplo.log
grep 'Failed password' ~/secure-ejemplo.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
grep 'Failed password' ~/secure-ejemplo.log | grep -oE 'for (invalid user )?[a-z]+' | awk '{print $NF}' | sort | uniq -c | sort -rn
```
Salida esperada (aprox.; los números varían por `shuf`):
```
Sep  3 01:11:01 rhel01 sshd[6001]: Failed password for invalid user root from 185.220.101.4 port 40001 ssh2
Sep  3 02:12:02 rhel01 sshd[6002]: Failed password for student from 10.0.2.2 port 40002 ssh2
Sep  3 03:13:03 rhel01 sshd[6003]: Failed password for invalid user admin from 45.33.12.9 port 40003 ssh2
     10 185.220.101.4
      8 45.33.12.9
      7 200.46.30.77
      5 10.0.2.2
     10 root
      8 admin
      7 oracle
      5 student
```
Qué observar: `$(NF-3)` es "el cuarto campo desde el final" (la IP está siempre antes de `port N ssh2`), lo que funciona aunque el usuario sea `invalid user x` o solo `x`. Con esta lista se decide: bloquear la IP en el firewall (`firewall-cmd --add-rich-rule='rule family=ipv4 source address=185.220.101.4 drop'`), revisar si `root` tiene login por SSH (no debería, Día 8), y activar `fail2ban` o `faillock`. En el journal, lo mismo: `journalctl _COMM=sshd -p info | grep -c 'Invalid user'`.

4. Limpieza del usuario de prueba (queda la regla de auditoría):
```bash
sudo userdel -r prueba_audit
sudo ausearch -k passwd_changes -ts recent -i | grep -c 'comm=userdel'
```
Salida esperada: `2` o más (userdel también tocó `passwd`, `shadow` y `group`).

- **Checkpoint:** pegar en el chat la salida de:
```bash
sudo auditctl -l | grep -c passwd_changes; sudo grep -c 'Invalid user' /var/log/secure; grep -c 'Failed password' ~/secure-ejemplo.log
```
Esperado: `3`, `3` (o más), `30`.

---

## Bloque 3 — Recuperación básica (30 min antes del descanso + 15 min después)

### Conceptos (10 min)

**El proceso de arranque, en cinco pasos.** Saber en qué paso se detuvo el servidor es la mitad del diagnóstico.

```text
1 Firmware (UEFI o BIOS)   busca el cargador: /boot/efi/EFI/redhat/ (UEFI) o el MBR (BIOS)
2 GRUB2                    muestra el menú (BLS: /boot/loader/entries/*.conf), carga kernel + initramfs
3 Kernel + initramfs       el initramfs trae drivers y LVM para encontrar /; monta /sysroot; pivota
4 systemd (PID 1)          lee /etc/fstab, arranca unidades hasta el default.target
5 default.target           multi-user.target (sin GUI) o graphical.target; getty y sshd listos
```

Dónde se atasca cada uno y qué se ve en pantalla:
- Paso 1–2: "No bootable device", `grub rescue>` → disco/orden de arranque/GRUB dañado (`grub2-install` en BIOS; en UEFI reinstalar `grub2-efi-x64`/`shim`).
- Paso 3: `Kernel panic - not syncing: VFS: Unable to mount root fs` → initramfs sin el driver o UUID de `root=` equivocado; elegir **otro kernel** en GRUB o regenerar con `dracut -f`.
- Paso 4: `You are in emergency mode` → casi siempre `/etc/fstab` (Lab 3.3); `Timed out waiting for device`.
- Paso 5: arranca pero un servicio falla → `systemctl --failed` (Bloque 1).

**Archivos de GRUB2 en RHEL 9.**
- `/etc/default/grub`: lo que uno edita (`GRUB_TIMEOUT`, `GRUB_CMDLINE_LINUX`, `GRUB_ENABLE_BLSCFG=true`).
- `/boot/grub2/grub.cfg`: **generado, no se edita**. Se regenera con `grub2-mkconfig -o /boot/grub2/grub.cfg`. En RHEL 9 ese es el destino **también en UEFI**: `/boot/efi/EFI/redhat/grub.cfg` es un archivo corto que redirige (`configfile`) al de `/boot/grub2/`. Verificar en cada VM con `sudo cat /boot/efi/EFI/redhat/grub.cfg` (si existe): si se ve el menú completo en vez de 3–4 líneas, esa VM usa el esquema antiguo y hay que regenerar ahí. En RHEL 8 con UEFI el destino era ese archivo; es una diferencia frecuente en tutoriales viejos.
- `/boot/loader/entries/*.conf`: una entrada por kernel (BootLoaderSpec). Las administra **`grubby`**: `--default-kernel`, `--info=ALL`, `--set-default`, `--args`/`--remove-args`. Editar estos archivos a mano es posible, pero `grubby` es la herramienta del examen.
- `/boot/grub2/grubenv`: variables (`saved_entry`, `menu_auto_hide`); `grub2-editenv list`.

**Targets de rescate.** Ambos requieren la **contraseña de root** y se usan desde la consola:
- `rescue.target` (el antiguo *single user*): monta todos los filesystems de `/etc/fstab`, arranca lo básico (sin red, sin sshd) y pide la contraseña de root. Para arreglar un servicio que impide el arranque normal.
- `emergency.target`: lo mínimo: `/` montado **solo lectura**, nada más. Es donde cae systemd solo cuando falla un montaje de `fstab`. Ahí siempre: `mount -o remount,rw /` antes de editar.
- Se puede ir en caliente (`systemctl isolate rescue.target`), elegir al arrancar (en GRUB, `systemd.unit=rescue.target` en la línea `linux`) o fijar el default (`systemctl set-default multi-user.target`).

**Cuando no se tiene la contraseña de root.** El acceso físico (o a la consola de la VM) equivale a root: por eso los servidores de producción llevan contraseña en GRUB y el centro de datos tiene puerta con llave. El procedimiento de RHEL 9 es `rd.break`: detener el arranque **dentro del initramfs**, antes de que el sistema real tome el control, cuando el disco ya está accesible en `/sysroot`. Ahí se cambia la contraseña y se pide un **relabel** de SELinux, porque `passwd` reescribe `/etc/shadow` sin que la política esté cargada y el archivo quedaría con contexto incorrecto (y nadie podría entrar).

**Red de seguridad.** En una VM, el snapshot del hipervisor es la recuperación más rápida que existe: antes de tocar GRUB, `fstab` o el kernel, snapshot. En físico, el equivalente es tener probado el procedimiento de hoy **antes** de necesitarlo. Y **Cockpit** (`https://IP:9090`, `systemctl enable --now cockpit.socket`) da una consola web con terminal, logs, servicios y almacenamiento cuando alguien sin experiencia en terminal tiene que actuar de emergencia.

### Lab 3.1 — GRUB visible, kernels y targets (5 min)

- **Objetivo:** garantizar que el menú de GRUB aparece en el arranque (imprescindible para los labs siguientes), leer la configuración con `grubby`, quitar `rhgb quiet` para ver el arranque completo y conocer los targets.

1. Ver la configuración actual:
```bash
cat /etc/default/grub
sudo grub2-editenv list
sudo grubby --default-kernel
sudo grubby --info=DEFAULT
ls /boot/loader/entries/
rpm -q kernel
```
Salida esperada (aprox.):
```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
saved_entry=3a7c...-5.14.0-503.14.1.el9_5.x86_64
boot_success=1
/boot/vmlinuz-5.14.0-503.14.1.el9_5.x86_64
index=0
kernel="/boot/vmlinuz-5.14.0-503.14.1.el9_5.x86_64"
args="ro crashkernel=... resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
root="/dev/mapper/rhel-root"
initrd="/boot/initramfs-5.14.0-503.14.1.el9_5.x86_64.img"
title="Red Hat Enterprise Linux (5.14.0-503.14.1.el9_5.x86_64) 9.5 (Plow)"
id="3a7c...-5.14.0-503.14.1.el9_5.x86_64"
3a7c...-5.14.0-503.14.1.el9_5.x86_64.conf  3a7c...-0-rescue.conf
kernel-5.14.0-503.14.1.el9_5.x86_64
```
Qué observar: hay una sola entrada de kernel más la de *rescue* (un initramfs genérico con todos los drivers, para cuando el normal no arranca). En UTM, `aarch64` en lugar de `x86_64` y `GRUB_TERMINAL_OUTPUT` puede no estar. Si la VM se actualizó el Día 4 con `dnf update`, puede haber **dos** kernels: `rpm -q kernel` los lista y GRUB muestra ambos; `dnf` conserva hasta 3 (`installonly_limit=3` en `/etc/dnf/dnf.conf`) y nunca borra el que está en uso.

2. Garantizar que el menú se vea 10 segundos y quitar `rhgb quiet` (para ver los mensajes del arranque en los labs de recuperación):
```bash
sudo grub2-editenv - unset menu_auto_hide
sudo sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=10/' /etc/default/grub
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grubby --update-kernel=ALL --remove-args="rhgb quiet"
sudo grubby --info=DEFAULT | grep args
if sudo test -f /boot/efi/EFI/redhat/grub.cfg; then sudo head -5 /boot/efi/EFI/redhat/grub.cfg; else echo "arranque BIOS: no hay /boot/efi"; fi
```
Salida esperada (aprox.):
```
Generating grub configuration file ...
Adding boot menu entry for UEFI Firmware Settings ...   (solo en UEFI)
done
args="ro crashkernel=... resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap"
search --no-floppy --root-dev-only --fs-uuid --set=dev 2f3a...
set prefix=($dev)/grub2
export $prefix
configfile $prefix/grub.cfg
```
Qué observar: `grubby --update-kernel=ALL --args="..."` / `--remove-args="..."` cambia los parámetros de **todas** las entradas BLS de forma persistente sin regenerar `grub.cfg`; `GRUB_CMDLINE_LINUX` en `/etc/default/grub` solo afecta a kernels que se instalen después (por eso se hacen las dos cosas: `sed` + `grub2-mkconfig` para el menú, `grubby` para los kernels ya instalados). Las 4 líneas con `configfile` confirman el esquema RHEL 9 en UEFI. En VirtualBox con BIOS no existe `/boot/efi` y se ve el mensaje del `else`.

3. Targets: leer y comprender sin cambiar el arranque:
```bash
systemctl get-default
systemctl list-units --type=target --state=active --no-legend | awk '{print $1}' | tr '\n' ' '; echo
systemctl list-dependencies rescue.target --no-pager | head -8
systemctl cat emergency.target | grep -E 'Requires|After|Description'
```
Salida esperada (aprox.):
```
multi-user.target
basic.target cryptsetup.target getty.target local-fs.target multi-user.target network-online.target network.target nfs-client.target paths.target remote-fs.target slices.target sockets.target sshd-keygen.target swap.target sysinit.target timers.target
rescue.target
● ├─rescue.service
● ├─sysinit.target
...
Description=Emergency Mode
Requires=emergency.service
After=emergency.service
```
Qué observar: `set-default` solo cambia el enlace `/etc/systemd/system/default.target`; `systemctl set-default graphical.target` en un servidor sin GUI arranca igual (el target existe, pero sin display manager). **Demo del instructor desde la consola** (quien lo repita perderá su sesión SSH): `sudo systemctl isolate rescue.target` → pide la contraseña de root → `systemctl list-units --type=service --state=running | wc -l` muestra unas pocas → `systemctl isolate multi-user.target` (o `systemctl default`) devuelve todo.

- **Checkpoint:** pegar en el chat la salida de:
```bash
grep GRUB_TIMEOUT /etc/default/grub; sudo grubby --info=DEFAULT | grep -c quiet; systemctl get-default
```
Esperado: `GRUB_TIMEOUT=10`, `0`, `multi-user.target`.

### Lab 3.2 — Restablecer la contraseña de root con `rd.break` (15 min)

- **Objetivo:** recuperar el acceso de root en una VM cuya contraseña "se perdió", con el procedimiento oficial de RHEL 9, y conocer la alternativa `init=/bin/bash`. Se trabaja **en la consola de la VM** (ventana de VirtualBox/UTM), no por SSH.

> Antes de empezar: snapshot `dia10-pre-rdbreak` (opcional, 30 segundos, evita sustos). Tener a mano la nueva contraseña que se pondrá (usar la misma que ya tenía root, para no desincronizar al grupo).

1. Reiniciar y detener el arranque en GRUB:
```bash
sudo systemctl reboot
```
En la consola, cuando aparece el menú de GRUB (ahora dura 10 s), pulsar **una flecha** para congelar la cuenta y luego **`e`** sobre la entrada del kernel. En UTM, si el menú no aparece, pulsar `Esc` repetidamente apenas arranca.

2. En el editor de GRUB, bajar hasta la línea que empieza con `linux` (`linuxefi` en algunas UEFI; en aarch64 es `linux`), ir al **final** de la línea (`Ctrl+E` o tecla `Fin`) y añadir un espacio y:
```text
rd.break
```
(Como ya se quitó `rhgb quiet`, se verá todo el arranque. Si en otra máquina siguen ahí, es buena idea borrarlos en esta misma línea.) Pulsar **Ctrl+X** para arrancar con esa línea, que **no queda guardada**.

3. El arranque se detiene dentro del initramfs con un prompt como:
```text
Press Enter for maintenance
(or press Control-D to continue):
switch_root:/#
```
Qué observar: ⚠️ **Verificar en la VM antes de la clase.** Si en lugar del prompt pide contraseña de root (ocurre en algunas versiones 9.x según la configuración del initramfs), pasar a la alternativa del paso 7. Aquí `/` es el initramfs (efímero) y el disco real está en `/sysroot`, **solo lectura**.

4. Remontar en escritura, entrar al sistema real y cambiar la contraseña:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
```
Salida esperada:
```
Changing password for user root.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```
Qué observar: dentro del `chroot`, `/etc/shadow` es el del disco. Si `passwd` se queja de la calidad de la contraseña (política del Día 8), **root puede ignorar el aviso**: repetirla y acepta.

5. Pedir el relabel de SELinux y salir (dos `exit`: uno del chroot, otro del initramfs; el arranque continúa solo):
```bash
touch /.autorelabel
exit
exit
```
Qué observar: el sistema arranca, aparece `*** Warning -- SELinux targeted policy relabel is required` con un contador de `*` y **reinicia solo** al terminar (1–5 min según el disco; en UTM emulado puede ser más). **Aquí empieza el descanso de 15 minutos**: las VMs terminan el relabel solas y el paso 6 se hace al volver. Sin `/.autorelabel`, `/etc/shadow` queda con contexto `unlabeled_t` y ningún usuario puede iniciar sesión: sería un segundo ticket. Alternativa más rápida y sin reinicio extra, para quien quiera probarla después de clase (el instructor decide si la enseña): dentro del chroot, `load_policy -i` antes de `passwd`, y `restorecon -v /etc/shadow` después; así no hace falta el relabel completo.

6. **Al volver del descanso**, verificar tras el reinicio y el relabel (desde la consola o por SSH):
```bash
su -
ls -Z /etc/shadow
journalctl -b -1 -n 3 --no-pager
exit
```
Salida esperada:
```
Password:
[root@rhel01 ~]# 
system_u:object_r:shadow_t:s0 /etc/shadow
(las últimas líneas del arranque del relabel: "Reached target Power-Off/Reboot")
```
Qué observar: la contraseña nueva funciona y `shadow_t` es el contexto correcto. Si `ls -Z` mostrara `unlabeled_t`, faltó el relabel: `sudo touch /.autorelabel && sudo reboot`.

7. **Alternativa `init=/bin/bash`** (útil si `rd.break` pide contraseña o si el initramfs no coopera). En GRUB, `e`, al final de la línea `linux` añadir `init=/bin/bash` (y quitar `rhgb quiet` si están), Ctrl+X. Aparece un prompt `bash-5.1#` con el disco real en `/` **solo lectura**:
```bash
mount -o remount,rw /
passwd root
touch /.autorelabel
exec /usr/lib/systemd/systemd
```
Qué observar: `exec /usr/lib/systemd/systemd` continúa el arranque normal desde ahí (el relabel se hará); si no funciona, `/usr/sbin/reboot -f` (el `-f` es obligatorio: no hay systemd que atienda un `reboot` normal). Con `init=/bin/bash` no hay `chroot` porque bash ya corre sobre el sistema real. Las dos técnicas valen en el examen; conviene dominar una y conocer la otra.

- **Checkpoint:** pegar en el chat la salida de:
```bash
ls -Z /etc/shadow; sudo grep -c '' /etc/shadow; uptime -s
```
Esperado: `system_u:object_r:shadow_t:s0 /etc/shadow`, el número de cuentas, y una hora de arranque de hace pocos minutos.

### Lab 3.3 — Reparar `/etc/fstab` desde emergency mode (12 min) — después del descanso

- **Objetivo:** provocar el fallo de arranque más común de un servidor (una línea inválida en `fstab` sin `nofail`), reconocer emergency mode, encontrar la causa con `journalctl -xb` y corregirla. Si el tiempo es corto, el instructor lo hace en demo y los participantes lo repetirán en el ticket T6.

1. Comprobar el estado sano y **provocar** el fallo (UUID inexistente, sin `nofail`):
```bash
findmnt --verify && echo "fstab OK"
echo "UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo mkdir -p /mnt/auditoria
findmnt --verify; echo "código: $?"
sudo mount -a
```
Salida esperada (aprox.):
```
fstab OK
UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0
/mnt/auditoria
   [W] unreachable on boot required source: UUID=deadbeef-0000-4000-8000-00000000c0de
0 parse errors, 0 errors, 1 warning
código: 0
mount: /mnt/auditoria: can't find UUID=deadbeef-0000-4000-8000-00000000c0de.
```
Qué observar: `findmnt --verify` y `mount -a` **ya lo avisan**: en la vida real, esta es la comprobación que evita el ticket. Hoy se ignora a propósito.

2. Reiniciar y observar la consola:
```bash
sudo systemctl reboot
```
Salida esperada en la consola (tras ~90 s de espera):
```
[  TIME ] Timed out waiting for device /dev/disk/by-uuid/deadbeef-0000-4000-8000-00000000c0de.
[DEPEND] Dependency failed for /mnt/auditoria.
[DEPEND] Dependency failed for Local File Systems.
...
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```
Qué observar: SSH no funciona (la red no arrancó). Escribir la contraseña de root en la consola.

3. Diagnosticar y corregir:
```bash
journalctl -xb -p err --no-pager | grep -iE 'mount|fstab|Dependency|Timed out' | tail -5
systemctl --failed --no-legend
mount -o remount,rw /
sed -i 's|^UUID=deadbeef.*|#&|' /etc/fstab
tail -2 /etc/fstab
systemctl daemon-reload
mount -a && findmnt --verify && echo "fstab OK"
systemctl default
```
Salida esperada (aprox.):
```
sep 04 09:20:10 rhel01 systemd[1]: dev-disk-by\x2duuid-deadbeef....device: Job ... timed out.
sep 04 09:20:10 rhel01 systemd[1]: Dependency failed for /mnt/auditoria.
sep 04 09:20:10 rhel01 systemd[1]: Dependency failed for Local File Systems.
mnt-auditoria.mount  loaded failed failed /mnt/auditoria
#UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0
fstab OK
```
Qué observar: `journalctl -xb` = este arranque (`-b`) con explicaciones (`-x`). El nombre de la unidad fallida (`mnt-auditoria.mount`) apunta directo a la línea de `fstab`. Sobre `mount -o remount,rw /`: en este caso concreto `/` suele estar ya en lectura-escritura (falló un montaje de datos, no la raíz) y el comando no hace daño; se ejecuta siempre por costumbre, porque en `rd.break` y en algunos fallos de la raíz sí es imprescindible y `vi` avisa con `Read-only file system`. En el examen puede usarse `vi /etc/fstab` para comentar o borrar la línea; `sed -i` es más rápido y menos propenso a errores de tecleo. `systemctl default` continúa hacia `multi-user.target` sin reiniciar; `reboot` también vale y además confirma que el arranque completo funciona.

4. Verificar por SSH y limpiar:
```bash
uptime; findmnt --verify | tail -1
sudo sed -i '/^#UUID=deadbeef/d' /etc/fstab
sudo rmdir /mnt/auditoria
sudo systemctl daemon-reload; systemctl --failed --no-legend | wc -l
```
Salida esperada: `0 parse errors, 0 errors, 0 warnings` y `0`.

**Menciones para la libreta (2 min):**
- **Kernel que no arranca:** en GRUB, elegir la entrada anterior (o la `rescue`); una vez dentro, `sudo grubby --set-default /boot/vmlinuz-VERSION-BUENA` y, si el initramfs está dañado, `sudo dracut -f /boot/initramfs-$(uname -r).img $(uname -r)` (`dracut -f` a secas regenera el del kernel en uso). `dnf install kernel` siempre deja el anterior: por eso nunca se queda uno sin kernel de respaldo. `lsinitrd | head` lista el contenido de un initramfs.
- **Contraseña de GRUB** (`grub2-setpassword`) para que `e` pida credenciales: obligatoria en producción, contraproducente en el laboratorio.
- **Cockpit:** `sudo dnf install -y cockpit` (en la instalación "Server" mínima no viene), `sudo systemctl enable --now cockpit.socket` y `https://192.168.56.10:9090` desde el navegador del equipo propio. El servicio `cockpit` viene abierto de fábrica en `public`, pero la red host-only se evalúa en `internal`: comprobar con `sudo firewall-cmd --zone=internal --list-services` y, si falta, `sudo firewall-cmd --permanent --zone=internal --add-service=cockpit && sudo firewall-cmd --reload`. Da terminal, logs, servicios, almacenamiento y red desde el navegador: el "plan B" para el técnico que no domina la terminal.

- **Checkpoint:** pegar en el chat la salida de:
```bash
findmnt --verify | tail -1; grep -c auditoria /etc/fstab; systemctl is-system-running
```
Esperado: `0 parse errors, 0 errors, 0 warnings`, `0`, `running` (o `degraded` si algún servicio menor falló: revisar con `systemctl --failed`).

---

## Bloque 4 — Laboratorio integrador: construir `web01` (20 min)

### Conceptos (3 min)

El servidor `web01` es el resultado de todo el curso: identidad (hostname), personas (grupo `sistemas`, `dev01`, `dev02`), una carpeta compartida con setgid, un sitio web en Apache publicado por el firewall con SELinux correcto, un volumen LVM dedicado a respaldos con `backup.sh` en cron, un export NFS y un contenedor rootless como servicio. Se construye con un script para que **los cuatro participantes tengan exactamente el mismo servidor**; luego se verifica con otro script que servirá también para comprobar los tickets. Antes de romper nada: **snapshot `dia10-pre-romper`**.

### Lab 4.1 — `construir-web01.sh` y `verificar-web01.sh` (17 min)

- **Objetivo:** dejar `web01` completo y verificado, y tomar el snapshot de seguridad.

1. Crear el script de construcción (el instructor lo pega en el chat; es idempotente: puede ejecutarse varias veces):
```bash
cat > ~/construir-web01.sh <<'EOF'
#!/bin/bash
# construir-web01.sh - Día 10. Deja la VM como el servidor web01 del laboratorio integrador.
# Ejecutar: sudo bash construir-web01.sh   (idempotente)
set -u
[[ $EUID -eq 0 ]] || { echo "Ejecutar con sudo"; exit 1; }
CLAVE='Pgn2026!lab'
ok() { echo "  [OK] $*"; }
# El Dia 8 dejo la red host-only 192.168.56.0/24 como origen de la zona "internal"
# y el NAT en "public": todo lo que se publique va en LAS DOS zonas.
abrir() {   # abrir --add-service=http  |  abrir --add-port=8085/tcp
  firewall-cmd --permanent "$1" >/dev/null 2>&1
  firewall-cmd --permanent --zone=internal "$1" >/dev/null 2>&1
}

echo "== 1. Identidad"
hostnamectl set-hostname web01.lab.local
grep -q 'web01' /etc/hosts || sed -i 's/rhel01.lab.local rhel01/web01.lab.local web01 rhel01.lab.local rhel01/' /etc/hosts
grep -q 'web01' /etc/hosts || echo "127.0.0.1 web01.lab.local web01" >> /etc/hosts
ok "hostname $(hostnamectl --static)"

echo "== 2. Grupo sistemas, dev01, dev02, /srv/compartido"
getent group sistemas >/dev/null || groupadd -g 3001 sistemas
for u in dev01 dev02; do
  id "$u" &>/dev/null || useradd -m -c "Desarrollador $u" -G sistemas "$u"
  usermod -aG sistemas "$u"; usermod -s /bin/bash -U "$u" 2>/dev/null
  echo "$u:$CLAVE" | chpasswd
  faillock --user "$u" --reset 2>/dev/null
done
mkdir -p /srv/compartido
chown root:sistemas /srv/compartido; chmod 2770 /srv/compartido
ok "$(getent group sistemas) ; $(stat -c '%A %U:%G' /srv/compartido)"

echo "== 3. Apache en 80 con DocumentRoot /web"
rpm -q httpd >/dev/null || dnf install -y httpd >/dev/null
rpm -q policycoreutils-python-utils >/dev/null || dnf install -y policycoreutils-python-utils >/dev/null
mkdir -p /web
echo "<h1>Servidor web01 - Procuraduria General de la Nacion</h1>" > /web/index.html
chmod 755 /web; chmod 644 /web/index.html
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?" 2>/dev/null || true
restorecon -Rv /web >/dev/null
sed -i 's/^Listen .*/Listen 80/' /etc/httpd/conf/httpd.conf
cat > /etc/httpd/conf.d/00-default.conf <<'CONF'
# Sitio por defecto de web01 (laboratorio integrador Dia 10)
<VirtualHost *:80>
    ServerName web01.lab.local
    ServerAlias web01 rhel01 localhost
    DocumentRoot /web
</VirtualHost>
<Directory "/web">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
CONF
apachectl configtest 2>&1 | tail -1
systemctl enable --now httpd >/dev/null 2>&1; systemctl restart httpd
abrir --add-service=http; firewall-cmd --reload >/dev/null
ok "httpd $(systemctl is-active httpd)/$(systemctl is-enabled httpd) ; HTTP $(curl -s -o /dev/null -w '%{http_code}' http://localhost/) ; http en zonas: public=$(firewall-cmd --list-services | grep -cw http) internal=$(firewall-cmd --zone=internal --list-services | grep -cw http)"

echo "== 4. /backups en LVM + backup.sh en cron diario"
if ! mountpoint -q /backups; then
  if [[ -e /dev/vg_datos/lv_backups ]]; then
    mkdir -p /backups
    grep -q ' /backups ' /etc/fstab || echo "/dev/mapper/vg_datos-lv_backups  /backups  ext4  defaults,nofail  0 0" >> /etc/fstab
    systemctl daemon-reload; mount -a
  elif vgs vg_datos &>/dev/null && lvcreate -y -n lv_backups -L 1G vg_datos >/dev/null 2>&1; then
    mkfs.ext4 -q /dev/vg_datos/lv_backups
    mkdir -p /backups
    grep -q ' /backups ' /etc/fstab || echo "/dev/mapper/vg_datos-lv_backups  /backups  ext4  defaults,nofail  0 0" >> /etc/fstab
    systemctl daemon-reload; mount -a
  else
    mkdir -p /backups; echo "  [AVISO] sin LV para /backups: queda en la raiz (T3 usara un archivo pequeno)"
  fi
fi
cat > /usr/local/bin/backup.sh <<'SCRIPT'
#!/bin/bash
# backup.sh - respalda un directorio en tar.gz con retencion de 7 dias (Dia 7, version web01)
# Uso: backup.sh [ORIGEN] [DESTINO]   Salida: 0 ok, 1 origen no existe, 2 fallo de tar
set -uo pipefail
ORIGEN="${1:-/home/student/empresa}"; DESTINO="${2:-/backups}"
FECHA=$(date +%F-%H%M); NOMBRE=$(basename "$ORIGEN")
ARCHIVO="$DESTINO/${NOMBRE}-${FECHA}.tar.gz"
log() { echo "$(date '+%F %T') $*"; logger -t backup "$*"; }
[[ -d "$ORIGEN" ]] || { log "ERROR: el origen $ORIGEN no existe"; exit 1; }
mkdir -p "$DESTINO"
if tar -czf "$ARCHIVO" -C "$(dirname "$ORIGEN")" "$NOMBRE" 2>/dev/null; then
    log "OK: creado $ARCHIVO ($(du -h "$ARCHIVO" | cut -f1))"
else
    log "ERROR: tar fallo al crear $ARCHIVO ($(df -h "$DESTINO" | awk 'NR==2{print $5}') usado en $DESTINO)"
    rm -f "$ARCHIVO"; exit 2
fi
BORRADOS=$(find "$DESTINO" -name "${NOMBRE}-*.tar.gz" -mtime +7 -print -delete | wc -l)
log "Retencion: $BORRADOS respaldo(s) de mas de 7 dias eliminado(s)"
SCRIPT
chmod 755 /usr/local/bin/backup.sh
mkdir -p /home/student/empresa/documentos; chown -R student:student /home/student/empresa
cat > /etc/cron.d/backup-empresa <<'CRON'
# Respaldo diario de la carpeta institucional a /backups (laboratorio web01)
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
30 23 * * * root /usr/local/bin/backup.sh /home/student/empresa /backups >> /var/log/backup.log 2>&1
CRON
/usr/local/bin/backup.sh /home/student/empresa /backups >> /var/log/backup.log 2>&1
ok "$(df -h /backups | awk 'NR==2{print $6, $5}') ; $(tail -1 /var/log/backup.log)"

echo "== 5. NFS exportando /srv/nfs/compartido"
rpm -q nfs-utils >/dev/null || dnf install -y nfs-utils >/dev/null
mkdir -p /srv/nfs/compartido; chown student:student /srv/nfs/compartido
echo "Compartido NFS de web01 - $(date +%F)" > /srv/nfs/compartido/LEAME.txt
grep -q '^/srv/nfs/compartido' /etc/exports || echo "/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server >/dev/null 2>&1
exportfs -ra
for s in nfs rpc-bind mountd; do abrir --add-service="$s"; done
firewall-cmd --reload >/dev/null
ok "$(showmount -e localhost | tail -1)"

echo "== 6. Contenedor portal (Quadlet de student, Dia 9)"
if curl -s -o /dev/null -w '%{http_code}' http://localhost:8085/ | grep -q 200; then
  ok "portal responde en 8085 ; linger: $(ls /var/lib/systemd/linger/ 2>/dev/null | tr '\n' ' ')"
else
  echo "  [PENDIENTE] portal no responde en 8085: crearlo como student (ver paso 2 del Lab 4.1)"
fi
loginctl enable-linger student
abrir --add-port=8085/tcp; firewall-cmd --reload >/dev/null

echo "== 7. Red host-only (perfil lab)"
if nmcli -g NAME connection show | grep -qx lab; then
  nmcli connection modify lab connection.autoconnect yes; nmcli connection up lab >/dev/null 2>&1
  ok "lab: $(nmcli -g IP4.ADDRESS connection show lab)"
else
  echo "  [AVISO] no existe el perfil 'lab' (Dia 5): T8 no aplicara"
fi
echo "== Listo. Ahora: sudo bash verificar-web01.sh"
EOF
sudo bash ~/construir-web01.sh
```
Salida esperada (aprox.):
```
== 1. Identidad
  [OK] hostname web01.lab.local
== 2. Grupo sistemas, dev01, dev02, /srv/compartido
  [OK] sistemas:x:3001:ana,carlos,dev01,dev02 ; drwxrws--- root:sistemas
== 3. Apache en 80 con DocumentRoot /web
Syntax OK
  [OK] httpd active/enabled ; HTTP 200 ; http en zonas: public=1 internal=1
== 4. /backups en LVM + backup.sh en cron diario
  [OK] /backups 2% ; 2026-09-04 09:40:12 OK: creado /backups/empresa-2026-09-04-0940.tar.gz (4.0K)
== 5. NFS exportando /srv/nfs/compartido
  [OK] /srv/nfs/compartido 192.168.56.0/24
== 6. Contenedor portal (Quadlet de student, Dia 9)
  [OK] portal responde en 8085 ; linger: student
== 7. Red host-only (perfil lab)
  [OK] lab: 192.168.56.10/24
== Listo. Ahora: sudo bash verificar-web01.sh
```
Qué observar: `hostnamectl` cambia el nombre al instante; el prompt lo mostrará al abrir una nueva sesión. El script usa `2>/dev/null || true` en `semanage fcontext -a` porque la regla ya existe desde el Día 8 (`already defined` no es un error aquí). La función `abrir()` publica cada servicio o puerto en **las dos zonas** (`public` para el NAT, `internal` para la red host-only, tal como se decidió el Día 8): es el error número uno del curso —"desde `localhost:8080` carga y desde `192.168.56.10` no"— y por eso está automatizado.

2. Solo si el paso 6 dijo `[PENDIENTE]` (el contenedor `portal` del Reto del Día 9 no existe), crearlo **como student**, sin `sudo`:
```bash
mkdir -p ~/portal ~/.config/containers/systemd
echo "<h1>Portal PGN</h1>" > ~/portal/index.html
cat > ~/.config/containers/systemd/portal.container <<'EOF'
[Unit]
Description=Portal PGN en contenedor
After=network-online.target

[Container]
Image=registry.access.redhat.com/ubi9/httpd-24
ContainerName=portal
PublishPort=8085:8080
Volume=/home/student/portal:/var/www/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user start portal.service
sleep 3; curl -s http://localhost:8085/
```
Salida esperada: `<h1>Portal PGN</h1>`. (El `pull` de la imagen tarda 1–2 min si no estaba descargada.)

3. Script de verificación (el mismo que se usará para evaluar los tickets):
```bash
cat > ~/verificar-web01.sh <<'EOF'
#!/bin/bash
# verificar-web01.sh - Día 10. Comprueba los 8 puntos del servidor web01. Ejecutar con sudo.
# Cada linea: T# OK/FALLA descripcion. Codigo de salida = numero de fallas.
[[ $EUID -eq 0 ]] || { echo "Ejecutar con sudo"; exit 1; }
fallas=0
chk() { if eval "$2" >/dev/null 2>&1; then echo "T$1 OK    $3"; else echo "T$1 FALLA $3"; fallas=$((fallas+1)); fi; }
chk 1 'systemctl is-active httpd && systemctl is-enabled httpd && firewall-cmd --list-services | grep -qw http && firewall-cmd --zone=internal --list-services | grep -qw http && curl -sf http://localhost/ | grep -q web01' \
      "Apache activo+habilitado, http en firewall (public e internal), index de web01 responde"
chk 2 'sudo -u dev01 touch /srv/compartido/.prueba_dev01 && rm -f /srv/compartido/.prueba_dev01 && [ "$(stat -c %A /srv/compartido)" = drwxrws--- ]' \
      "dev01 escribe en /srv/compartido (2770 root:sistemas)"
chk 3 '[ "$(df --output=pcent /backups | tail -1 | tr -dc 0-9)" -lt 90 ] && /usr/local/bin/backup.sh /home/student/empresa /backups' \
      "/backups con espacio y backup.sh termina OK"
chk 4 'passwd -S dev02 | grep -qw P && getent passwd dev02 | grep -q /bin/bash && ! faillock --user dev02 | grep -qE "V$"' \
      "dev02 desbloqueado, con shell /bin/bash y sin faillock"
chk 5 '[ "$(curl -s -o /dev/null -w %{http_code} http://localhost/)" = 200 ] && ls -Z /web/index.html | grep -q httpd_sys_content_t' \
      "index.html con contexto httpd_sys_content_t y HTTP 200"
chk 6 'findmnt --verify 2>&1 | grep -q "0 errors" && ! grep -qE "^[^#].*deadbeef|^[^#].*/mnt/respaldo_viejo" /etc/fstab' \
      "fstab sin lineas invalidas (findmnt --verify)"
chk 7 '[ -e /var/lib/systemd/linger/student ] && firewall-cmd --zone=internal --list-ports | grep -qw 8085/tcp && curl -sf http://localhost:8085/ | grep -q Portal' \
      "linger de student, 8085 abierto en internal y contenedor portal respondiendo"
chk 8 'nmcli -t -f NAME,DEVICE connection show --active | grep -q "^lab:" && [ "$(nmcli -g connection.autoconnect connection show lab)" = yes ]' \
      "perfil lab activo y con autoconnect"
echo "NFS: $(showmount -e localhost 2>/dev/null | tail -1)   hostname: $(hostname)"
echo "Fallas: $fallas"; exit $fallas
EOF
sudo bash ~/verificar-web01.sh
```
Salida esperada:
```
T1 OK    Apache activo+habilitado, http en firewall (public e internal), index de web01 responde
T2 OK    dev01 escribe en /srv/compartido (2770 root:sistemas)
T3 OK    /backups con espacio y backup.sh termina OK
T4 OK    dev02 desbloqueado, con shell /bin/bash y sin faillock
T5 OK    index.html con contexto httpd_sys_content_t y HTTP 200
T6 OK    fstab sin lineas invalidas (findmnt --verify)
T7 OK    linger de student, 8085 abierto en internal y contenedor portal respondiendo
T8 OK    perfil lab activo y con autoconnect
NFS: /srv/nfs/compartido 192.168.56.0/24   hostname: web01.lab.local
Fallas: 0
```
Qué observar: cada línea del verificador corresponde a un ticket. La comprobación **externa** (navegador del equipo propio en `http://192.168.56.10/` y `http://192.168.56.10:8085/`, o `http://localhost:8080/` por port forwarding) la hace cada participante ahora, para saber cómo se ve cuando funciona.

4. **Snapshot `dia10-pre-romper`** (con la VM encendida está bien en VirtualBox; en UTM, apagar, snapshot, encender). Es la red de seguridad del reto: quien se atasque más de 15 minutos en un ticket puede restaurar y seguir con el siguiente.

- **Checkpoint:** pegar en el chat la salida de:
```bash
sudo bash ~/verificar-web01.sh | tail -1; hostname; curl -s http://localhost/ | head -1
```
Esperado: `Fallas: 0`, `web01.lab.local`, `<h1>Servidor web01 - ...</h1>`.

---

## Reto integrador (70 min)

El instructor comparte `romper.sh` por el chat. Cada participante lo ejecuta **desde la consola de la VM** (ventana de VirtualBox/UTM), como root, y **reinicia** cuando el script lo indique. Después recibe los ocho tickets. Desde ese momento no hay pistas: solo `man`, los cheatsheets de los diez días y el método.

```bash
sudo bash romper.sh              # los 8 tickets (T6 exige reiniciar: se trabaja primero en consola)
sudo bash romper.sh --sin-fstab  # 7 tickets, sin tocar el arranque (recomendado si el Lab 3.3 no se hizo)
```

**Tickets (texto tal como llega a la mesa de ayuda):**

```text
OBLIGATORIOS
T1  "La página web de la Procuraduría no carga desde las computadoras de la oficina.
     Ayer sí funcionaba. Es urgente, la publica Comunicación Social."
T2  "Soy dev01. Desde esta mañana no puedo guardar nada en la carpeta compartida
     /srv/compartido. Mi compañero dev02 tampoco. Dice 'Permiso denegado'."
T3  "El respaldo de anoche falló. En el correo de error dice algo de 'No space left'.
     Necesitamos que el respaldo vuelva a funcionar hoy y que se ejecute solo cada noche."

INTERMEDIOS
T4  "dev02 no puede iniciar sesión en el servidor. Probó con su contraseña varias
     veces y nada. Está seguro de que es la correcta."
T5  "Después de que 'arreglaron' la web, ahora muestra 'Forbidden' (403). El archivo
     index.html está ahí, lo acabo de ver, y tiene permisos de lectura."

BONUS
T6  "Después del reinicio de mantenimiento el servidor no levanta: la pantalla pide
     una contraseña de mantenimiento. Nadie puede entrar por SSH."
T7  "El portal en contenedor (puerto 8085) dejó de responder desde el reinicio.
     Antes arrancaba solo."
T8  "Desde la red del laboratorio (192.168.56.x) el servidor ya no responde a nada,
     ni ping. Por la otra red sí."

Entregable por ticket (4 líneas): qué estaba mal · cómo lo encontré (comando) ·
cómo lo corregí (comando) · cómo lo verifiqué (desde donde lo ve el usuario).
Prohibido: setenforce 0, systemctl stop firewalld, chmod 777, chcon como solución
final, restaurar el snapshot como "solución" (sí como reinicio del intento).
```

**Orden sugerido para el participante** (no se entrega con los tickets; el instructor lo dice en voz alta): si el servidor está en emergency mode, T6 va primero por fuerza. Luego T1 → T5 (aparece al resolver T1) → T2 → T3 → T4 → T7 → T8. Ejecutar `sudo bash ~/verificar-web01.sh` cuantas veces haga falta: es el marcador.

**Rúbrica de evaluación** (por ticket, 4 puntos; se considera resuelto solo con verificación externa):

| Criterio | 1 punto si... | 0 puntos si... |
|---|---|---|
| Qué estaba mal | Nombra la causa real (por ejemplo "httpd deshabilitado **y** http fuera del firewall") | Describe el síntoma ("la web no cargaba") |
| Cómo lo encontró | Cita el comando que lo delató (`systemctl status`, `firewall-cmd --list-all`, `ls -Z`, `df -h`, `passwd -S`, `journalctl -xb`...) | "Probando cosas" |
| Cómo lo corrigió | Corrección mínima y **persistente** (`enable --now`, `--permanent` + `--reload`, `restorecon`, `passwd -u` + `usermod -s`, `nmcli`) | Corrección volátil o prohibida |
| Cómo lo verificó | Desde el punto de vista del usuario: navegador del host, `su - dev01` + `touch`, `backup.sh` con código 0, reinicio cuando aplica | Solo `curl localhost` o "ya está" |

Puntaje: T1–T3 obligatorios (12 puntos), T4–T5 (8), T6–T8 bonus (12). Aprobado: 10 de 12 en obligatorios. El instructor anota el tiempo por ticket: el objetivo formativo es que T1–T3 salgan en menos de 25 minutos.

### Script `romper.sh` (para el instructor)

```bash
#!/bin/bash
# romper.sh — Día 10, laboratorio integrador. Ejecutar como root DESDE LA CONSOLA de la VM.
#   sudo bash romper.sh              -> 8 tickets; termina pidiendo reiniciar (T6)
#   sudo bash romper.sh --sin-fstab  -> 7 tickets, sin tocar el arranque
# Registro (solo instructor): /root/romper-dia10.log
set -u
[[ $EUID -eq 0 ]] || { echo "Ejecutar con sudo"; exit 1; }
SIN_FSTAB=0; [[ "${1:-}" == "--sin-fstab" ]] && SIN_FSTAB=1
LOG=/root/romper-dia10.log
echo "==== $(date '+%F %T') romper.sh (sin_fstab=$SIN_FSTAB)" >> "$LOG"

# T1: web no carga desde afuera (servicio detenido y deshabilitado + firewall cerrado
#     en LAS DOS zonas: public para el NAT, internal para la red host-only)
systemctl disable --now httpd >/dev/null 2>&1
firewall-cmd --permanent --remove-service=http >/dev/null 2>&1
firewall-cmd --permanent --zone=internal --remove-service=http >/dev/null 2>&1
firewall-cmd --reload >/dev/null
echo "T1 httpd disabled+stopped; http fuera de public e internal" >> "$LOG"

# T2: dev01 no puede escribir en /srv/compartido
chown root:root /srv/compartido; chmod 755 /srv/compartido
echo "T2 /srv/compartido root:root 755" >> "$LOG"

# T3: backup falló por disco lleno (archivo oculto). Si /backups es un montaje propio se llena
#     del todo (incluida la reserva de root); si no, solo un archivo pequeño para no llenar /.
if mountpoint -q /backups; then
  dd if=/dev/zero of=/backups/.cache_old.img bs=1M status=none 2>/dev/null
  echo "T3 /backups lleno con /backups/.cache_old.img" >> "$LOG"
else
  fallocate -l 200M /backups/.cache_old.img
  echo "T3 /backups NO es montaje propio: solo .cache_old.img de 200M (no llena /)" >> "$LOG"
fi
/usr/local/bin/backup.sh /home/student/empresa /backups >> /var/log/backup.log 2>&1
rm -f /etc/cron.d/backup-empresa
echo "T3 backup.sh ejecutado (ERROR en /var/log/backup.log); /etc/cron.d/backup-empresa borrado" >> "$LOG"

# T4: dev02 bloqueado y sin shell
passwd -l dev02 >/dev/null
usermod -s /sbin/nologin dev02
echo "T4 dev02: passwd -l + shell /sbin/nologin" >> "$LOG"

# T5: index.html con contexto de /root (aparece al resolver T1)
echo "<h1>Servidor web01 - Procuraduria General de la Nacion</h1>" > /root/index.html
mv -f /root/index.html /web/index.html
chmod 644 /web/index.html
echo "T5 /web/index.html con admin_home_t ($(ls -Z /web/index.html | awk '{print $1}'))" >> "$LOG"

# T7: contenedor portal no vuelve tras reboot (linger apagado) y se detiene ahora
UID_S=$(id -u student)
runuser -u student -- env XDG_RUNTIME_DIR="/run/user/$UID_S" \
        DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$UID_S/bus" \
        systemctl --user stop portal.service >/dev/null 2>&1 \
  || runuser -l student -c 'podman stop portal' >/dev/null 2>&1
loginctl disable-linger student
echo "T7 portal detenido; linger de student deshabilitado" >> "$LOG"

# T8: red host-only sin IP (perfil lab abajo y sin autoconnect). No afecta la ruta NAT (SSH 2222).
if nmcli -g NAME connection show | grep -qx lab; then
  nmcli connection modify lab connection.autoconnect no
  nmcli connection down lab >/dev/null 2>&1
  echo "T8 perfil lab down + autoconnect no" >> "$LOG"
fi

# T6 (ÚLTIMO): línea inválida en fstab sin nofail -> emergency mode en el próximo arranque
if [[ $SIN_FSTAB -eq 0 ]]; then
  echo "UUID=deadbeef-1111-4222-8333-000000000bad  /mnt/respaldo_viejo  xfs  defaults  0 0" >> /etc/fstab
  mkdir -p /mnt/respaldo_viejo
  echo "T6 fstab con UUID inexistente sin nofail" >> "$LOG"
  echo
  echo ">>> Fallas aplicadas. AHORA REINICIE: sudo systemctl reboot  (trabaje desde la CONSOLA)"
else
  echo
  echo ">>> Fallas aplicadas (sin T6). Puede seguir por SSH. Suerte."
fi
```

### Solución (para el instructor)

| Ticket | Diagnóstico esperado (comandos que lo delatan) | Corrección | Verificación |
|---|---|---|---|
| **T6** (primero si aplica) | Consola: `Give root password for maintenance`. `journalctl -xb -p err \| grep -iE 'mount\|Dependency'` → `Dependency failed for /mnt/respaldo_viejo`; `systemctl --failed` → `mnt-respaldo_viejo.mount` | `mount -o remount,rw /`; `vi /etc/fstab` (comentar/borrar la línea `deadbeef`); `systemctl daemon-reload`; `mount -a`; `findmnt --verify`; `systemctl default` o `reboot` | Arranca a `multi-user`, SSH vuelve; `findmnt --verify` → `0 errors`; `uptime` |
| **T1** | `systemctl status httpd` → `inactive (dead)` y `disabled`; tras `start`, `curl localhost` da 403 (→ T5) pero desde el host no carga; `sudo firewall-cmd --list-services` y `sudo firewall-cmd --zone=internal --list-services` sin `http`; `sudo firewall-cmd --get-active-zones` recuerda que 192.168.56.0/24 cae en `internal`; `sudo ss -tlnp \| grep :80` | `sudo systemctl enable --now httpd`; `sudo firewall-cmd --permanent --add-service=http`; `sudo firewall-cmd --permanent --zone=internal --add-service=http`; `sudo firewall-cmd --reload` | Navegador del host: `http://192.168.56.10/` (o `localhost:8080`) muestra "Servidor web01"; `systemctl is-enabled httpd` → `enabled`. **Quien solo abra `public` verá que `localhost:8080` funciona y `192.168.56.10` no: el ticket no está resuelto** |
| **T5** | `curl -I localhost` → 403; `ls -l /web/index.html` correcto (644); `ls -Z /web/index.html` → `admin_home_t`; `sudo ausearch -m AVC -ts recent \| grep index.html`; `sudo tail /var/log/httpd/error_log` → `Permission denied` | `sudo restorecon -v /web/index.html` (la regla fcontext de `/web` ya existe: `semanage fcontext -l \| grep '^/web'`) | `curl -s localhost/` muestra el `<h1>`; navegador del host; `ls -Z` → `httpd_sys_content_t` |
| **T2** | `su - dev01` → `touch /srv/compartido/x` → `Permission denied`; `ls -ld /srv/compartido` → `drwxr-xr-x root root`; `id dev01` sí está en `sistemas` | `sudo chown root:sistemas /srv/compartido && sudo chmod 2770 /srv/compartido` | `su - dev01 -c 'touch /srv/compartido/prueba && ls -l /srv/compartido'` → el archivo nace con grupo `sistemas` |
| **T3** | `tail /var/log/backup.log` → `ERROR: tar fallo ... (100% usado en /backups)`; `df -h /backups` → 100 %; `sudo du -sh /backups/*` no lo explica; `sudo ls -la /backups` o `sudo du -ah /backups \| sort -h \| tail -3` → `.cache_old.img`; `sudo lsof +L1` no muestra nada (no está abierto); `ls /etc/cron.d/` → falta `backup-empresa` | `sudo rm /backups/.cache_old.img`; recrear `/etc/cron.d/backup-empresa` (o `sudo crontab -e` con la línea `30 23 * * * /usr/local/bin/backup.sh /home/student/empresa /backups >> /var/log/backup.log 2>&1`) | `sudo /usr/local/bin/backup.sh /home/student/empresa /backups; echo $?` → `0`; `df -h /backups` bajo; `sudo cat /etc/cron.d/backup-empresa`; `journalctl -t backup -n 1` |
| **T4** | `su - dev02` → falla; `sudo passwd -S dev02` → `dev02 LK ...` (bloqueada); `getent passwd dev02` → `/sbin/nologin`; `sudo faillock --user dev02` (si probó muchas veces, además bloqueado por faillock); `sudo grep dev02 /var/log/secure` | `sudo passwd -u dev02`; `sudo usermod -s /bin/bash dev02`; `sudo faillock --user dev02 --reset` si aplica. (SSH como dev02 **no** debe funcionar: el Día 8 dejó `AllowUsers student`; eso no es parte del ticket y no se "arregla") | `su - dev02 -c 'id; echo $SHELL'` con la contraseña `Pgn2026!lab`; `passwd -S dev02` → `P` |
| **T7** | Como student: `podman ps -a` → `portal` `Exited`; `systemctl --user status portal` → `inactive`; `ls /var/lib/systemd/linger/` vacío; `loginctl show-user student -p Linger` → `Linger=no` | `sudo loginctl enable-linger student`; `systemctl --user start portal.service` | `curl -s localhost:8085/` → `Portal PGN`; navegador `http://192.168.56.10:8085/`; tras `sudo reboot`, sin abrir sesión, sigue respondiendo |
| **T8** | `ip -br a` sin `192.168.56.10`; `nmcli device status` → `enp0s8 disconnected`; `nmcli con show` → `lab` sin device; `nmcli -g connection.autoconnect con show lab` → `no` | `sudo nmcli connection modify lab connection.autoconnect yes && sudo nmcli connection up lab` | `ping -c 2 192.168.56.10` desde el host; `nmcli -f NAME,DEVICE con show --active`; sobrevive a `reboot` |

Verificación final de cada participante: `sudo bash ~/verificar-web01.sh` → `Fallas: 0`, más las capturas del navegador (`http://IP/` y `http://IP:8085/`).

**Errores que se verán (y qué decir):**
- Empezar por el firewall sin mirar `systemctl status`: "primero el servicio, después la puerta".
- En T1, abrir `http` solo en `public` y declararlo resuelto: desde el NAT carga y desde 192.168.56.10 no. Es la trampa de las dos zonas del Día 8; el verificador la detecta.
- Arreglar T1 con `systemctl start` y olvidar `enable`: el verificador lo detecta; en producción es el ticket que vuelve tras el siguiente reinicio.
- En T5, `chcon -t httpd_sys_content_t` funciona hoy: preguntar "¿y después de un relabel?"; pedir `restorecon`.
- En T3, borrar los `.tar.gz` legítimos porque `du -sh /backups/*` no muestra el oculto: recordar `ls -la` y `du -ah`.
- En T4, resolver `passwd -u` y declarar resuelto sin probar `su - dev02`: el shell `nologin` sigue ahí. Y probar 5 veces con contraseña errónea dispara faillock (Día 8): ahora hay tres causas.
- En T6, editar `fstab` con `/` en solo lectura (`vi` avisa `Read-only file system`): `mount -o remount,rw /`.
- En T7, `systemctl --user enable portal` → `is transient or generated`: con Quadlet no se hace enable; la causa es linger.
- En T8, `ip addr add 192.168.56.10/24 dev enp0s8` funciona hasta el reinicio: "persistente o no está resuelto".

---

## Cierre del curso (20 min)

### Puesta en común (6 min)
Cada participante cuenta **el ticket que más le costó** y qué comando lo destrabó. El instructor cierra con la idea del método: los ocho tickets se resuelven con la misma tabla del Bloque 1 aplicada en orden; ninguno requería un comando que no se hubiera visto en el curso.

### Mapa de lo visto contra la ficha técnica (4 min)

| Módulo | Tema | Día | Estado |
|---|---|---|---|
| RH124 M1 | Introducción a Linux, shell, navegación | 1, 2 | Practicado |
| RH124 M2 | Gestión de archivos, compresión, búsquedas | 2 | Practicado |
| RH124 M3 | Usuarios, grupos, permisos estándar y especiales | 3 | Practicado |
| RH124 M4 | Procesos, servicios, logs del sistema | 4 (y 10) | Practicado |
| RH124 M5 | Configuración IP, DNS, SSH | 5 | Practicado |
| RH124 M6 | RPM, DNF/YUM, repositorios | 4 (y 1, 2) | Practicado |
| RH124 M7 | Particiones, sistemas de archivos, montaje | 6 | Practicado |
| RH124 M8 | Firewall básico, SELinux introducción, buenas prácticas | 8 | Practicado |
| RH134 M1 | Bash scripting, variables, loops, automatización | 7 | Practicado |
| RH134 M2 | Cron, at, automatización empresarial | 7 | Practicado (cron, at, timers); Ansible en demo |
| RH134 M3 | NetworkManager, IP avanzada, troubleshooting de red | 5 | Practicado |
| RH134 M4 | HTTP, NFS, SMB, FTP | 9 | Practicado (HTTP, NFS, SMB); FTP en demo |
| RH134 M5 | SELinux avanzado, firewalld, hardening | 8 | Practicado; OpenSCAP en demo |
| RH134 M6 | LVM, Stratis, VDO, automontaje | 6, 9 | Practicado (LVM, Stratis, autofs); VDO mencionado |
| RH134 M7 | Podman, imágenes, contenedores | 9 | Practicado |
| RH134 M8 | Diagnóstico, logs avanzados, recuperación | 10 | Practicado |

### Objetivos del examen RHCSA EX200 (RHEL 9) y qué día los cubrió (4 min)

| Grupo de objetivos EX200 | Día(s) | Nota |
|---|---|---|
| Understand and use essential tools (shell, redirección, grep, tar, vim, man, permisos, enlaces) | 1, 2, 3 | Base de todo; el examen es 100 % terminal |
| Create simple shell scripts (condicionales, bucles, argumentos, códigos de salida) | 7 | `backup.sh`, `crear-usuarios.sh` |
| Operate running systems (boot, targets, `rd.break`, procesos, tuned, logs, journal persistente, `scp`/`rsync`) | 1, 4, 5, 10 | Hoy: interrupt boot, targets, journal |
| Configure local storage (particiones, LVM, fstab por UUID, swap) | 6 | `lvextend -r`, `nofail` |
| Create and configure file systems (XFS/ext4, NFS, autofs, setgid, ACL, `df`/`du`) | 3, 6, 9 | autofs el Día 9 |
| Deploy, configure and maintain systems (targets, cron/at, servicios, kernel/GRUB, dnf, timers) | 1, 4, 7, 10 | Hoy: `grubby`, kernels |
| Manage basic networking (nmcli, IP/gateway/DNS, hostname, firewalld) | 5, 8 | |
| Manage users and groups (useradd, passwd, chage, sudo) | 3, 8 | faillock/pwquality el Día 8 |
| Manage security (firewalld, SELinux contextos/puertos/booleanos, `ausearch`, ssh keys) | 5, 8, 9 | Diagnóstico de AVC |
| Manage containers (podman, registros, volúmenes, servicio systemd) | 9 | Quadlet y `generate systemd` |

### Cómo seguir (4 min)
- **Práctica diaria:** 20 minutos al día en la VM durante un mes valen más que un fin de semana entero. Recomendación concreta: **rehacer los 10 labs en dos semanas**, un día por sesión, desde el snapshot `dia01-fin`, sin mirar el material hasta atascarse.
- **Recursos oficiales:** documentación de producto RHEL 9 en https://docs.redhat.com (guías "Configuring basic system settings", "Managing storage devices", "Using SELinux", "Building, running, and managing containers"); base de conocimiento y herramientas (Product Life Cycle, CVE, Insights) en https://access.redhat.com; suscripción gratuita, imágenes y laboratorios interactivos en https://developers.redhat.com. Objetivos oficiales del examen: buscar "EX200 exam objectives" en redhat.com.
- **Red Hat Learning Subscription (RHLS):** acceso a los cursos oficiales RH124/RH134 en línea con laboratorios en la nube; **RH199** es la versión acelerada (5 días) de RH124+RH134 para quien ya trabajó con Linux, ideal para repasar antes del EX200.
- **Rutas después del RHCSA:** **RHCE (EX294)** con **RH294** (automatización con Ansible: el paso natural para administrar decenas de servidores institucionales); contenedores y **OpenShift** (DO180/DO280) si la institución mueve aplicaciones a plataformas de contenedores; **RH362** (Identity Management) para integrar con Active Directory.
- **Para el examen:** 3 horas, VM sin internet pero con `man` y `/usr/share/doc`; se califica por resultado tras reinicio. Practicar cada tarea hasta hacerla en menos de 5 minutos, siempre verificando tras `reboot`.

### Cierre operativo (2 min)
- `sudo bash ~/verificar-web01.sh` por última vez y **snapshot `dia10-fin`** con la VM apagada. Conservar `dia10-pre-romper` para repetir el reto.
- Entregar los cheatsheets de los 10 días en un solo PDF (tarea del instructor) y la lista de objetivos EX200 con el día que los cubre (tabla anterior).

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `systemctl --failed` / `systemctl status -l X` | Unidades fallidas / estado completo de un servicio |
| `journalctl -p err -b` / `journalctl -xe` | Errores de este arranque / últimas entradas con explicación |
| `journalctl -u X --since "1 hour ago"` | Log de un servicio en una ventana de tiempo |
| `journalctl --list-boots` / `-b -1` / `-k` | Arranques guardados / arranque anterior / mensajes del kernel |
| `journalctl _SYSTEMD_UNIT=x.service _PID=N _UID=N _COMM=cmd` | Filtros por campo (ver campos con `-o verbose`) |
| `journalctl -o verbose` / `-o json-pretty` / `-F CAMPO` | Ver todos los campos / exportar / listar valores de un campo |
| `journalctl --disk-usage` / `--vacuum-time=2weeks` | Tamaño del journal / borrar archivados por antigüedad |
| `/etc/systemd/journald.conf.d/*.conf` (`Storage=persistent`, `SystemMaxUse=`) | Persistencia y límite del journal |
| `sudo dmesg -T \| grep -iE 'oom\|error'` | Kernel: OOM killer, discos, drivers |
| `systemd-analyze time` / `blame` / `critical-chain` | Cuánto tardó el arranque y quién lo retrasó |
| `last` / `lastb` / `who` / `w` | Sesiones, reinicios, intentos fallidos, quién está ahora |
| `df -h` / `df -i` / `du -xsh /ruta/* \| sort -h` | Espacio / inodos / quién lo ocupa |
| `sudo lsof +L1` / `: > /proc/PID/fd/N` | Archivos borrados aún abiertos / liberar sin reiniciar |
| `sudo ss -tulpn` / `sudo lsof -i :80` | Qué escucha y qué proceso |
| `vmstat 1 5` / `iostat -x 1 3` / `sar -u 1 3` (sysstat) | CPU, memoria, swap, disco |
| `sudo strace -p PID -e trace=network` | En qué llamada está detenido un proceso |
| `rsyslogd -N1` | Validar `/etc/rsyslog.conf` y `/etc/rsyslog.d/*` |
| `local5.* /var/log/x.log` · `*.* @@IP:514` (TCP) · `@IP:514` (UDP) | Regla local / envío a colector |
| `module(load="imtcp")` + `input(type="imtcp" port="514" ruleset="r")` | Recibir logs remotos por TCP (con ruleset propio) |
| `semanage port -m -t syslogd_port_t -p tcp 514` (revertir: `-d -p tcp 514`) | Permitir a rsyslog enlazar 514/tcp (por defecto es `rsh_port_t`) |
| `local5.* stop` | Cortar la evaluación de un mensaje para que no se duplique en `/var/log/messages` |
| `logger -p local5.err -t app "msg"` / `logger -n IP -P 514 -T "msg"` | Probar reglas locales / enviar directo a un colector |
| `sudo logrotate -d /etc/logrotate.d/x` / `-f` | Simular / forzar una rotación |
| `-w /etc/passwd -p wa -k llave` en `/etc/audit/rules.d/*.rules` + `augenrules --load` | Regla persistente de auditd |
| `ausearch -k llave -ts recent -i` / `-ua UID -ts today` / `aureport --summary` | Buscar eventos de auditoría / resumen |
| `grep 'Failed password' /var/log/secure \| awk '{print $(NF-3)}' \| sort \| uniq -c \| sort -rn` | Intentos SSH fallidos por IP |
| `grubby --default-kernel` / `--info=ALL` / `--set-default /boot/vmlinuz-X` | Ver y elegir kernel por defecto |
| `grubby --update-kernel=ALL --args="x"` / `--remove-args="rhgb quiet"` | Parámetros del kernel persistentes |
| `grub2-mkconfig -o /boot/grub2/grub.cfg` (tras editar `/etc/default/grub`) | Regenerar GRUB (RHEL 9: mismo destino en BIOS y UEFI) |
| `systemctl get-default` / `set-default multi-user.target` / `isolate rescue.target` | Target por defecto / cambiar en caliente (consola) |
| GRUB `e` → `rd.break` → `mount -o remount,rw /sysroot; chroot /sysroot; passwd; touch /.autorelabel; exit; exit` | Restablecer contraseña de root |
| GRUB `e` → `init=/bin/bash` → `mount -o remount,rw /; passwd; touch /.autorelabel; exec /usr/lib/systemd/systemd` | Alternativa sin initramfs |
| Emergency: `journalctl -xb`, `mount -o remount,rw /`, editar fstab, `systemctl daemon-reload`, `mount -a`, `findmnt --verify`, `systemctl default` | Salir de emergency mode |
| `dracut -f` / `lsinitrd` | Regenerar / inspeccionar el initramfs |

---

## Notas para el instructor

### Preparar antes de la clase
- Restaurar `dia09-fin` en una VM de pruebas y ejecutar **todo el día de corrido, cronometrado**, incluidos los dos reinicios del Bloque 3 y el reto completo. Anotar cuánto tarda el relabel en cada hipervisor (en UTM emulado puede ser 5+ min).
- **Ensayar `rd.break` en UTM y en VirtualBox.** Con UEFI (UTM siempre; VirtualBox solo si se activó EFI) el menú de GRUB se ve distinto y la línea puede empezar por `linuxefi`. En UTM a veces hay que pulsar `Esc` apenas aparece el logo para ver el menú. Garantizar que aparezca: `sudo grub2-editenv - unset menu_auto_hide` y `GRUB_TIMEOUT=10` + `grub2-mkconfig` (Lab 3.1 lo hace en todas las VMs). Probar **las dos** técnicas (`rd.break` e `init=/bin/bash`): en algunas versiones 9.x `rd.break` pide contraseña o el prompt aparece con `Press Enter for maintenance`. Probar también la variante `load_policy -i` + `restorecon /etc/shadow` para decidir si se enseña.
- Verificar que `sudo cat /boot/efi/EFI/redhat/grub.cfg` en la VM UTM muestra el `configfile` (esquema RHEL 9). Si muestra el menú completo, ajustar el texto del Bloque 3 para esa VM.
- Probar `construir-web01.sh`, `verificar-web01.sh` y `romper.sh` (con y sin `--sin-fstab`) tres veces seguidas: construir → verificar (0 fallas) → romper → reiniciar → resolver todo → verificar (0 fallas). Confirmar que T3 llena `/backups` y que `backup.log` muestra el `ERROR`. Confirmar que T7 detiene el contenedor: el script intenta primero `runuser -u student -- env XDG_RUNTIME_DIR=... systemctl --user stop portal.service` y, si no hay `/run/user/1000` (nadie con sesión abierta), cae al `runuser -l student -c 'podman stop portal'`; verificar que al menos uno de los dos funciona en la VM.
- **Comprobar las dos zonas del firewall.** Desde el Día 8, `192.168.56.0/24` es origen de `internal` y el NAT entra por `public`. `construir-web01.sh` publica en las dos (función `abrir()`) y `romper.sh` cierra `http` en las dos: si no fuera así, T1 no se reproduciría (la web seguiría cargando desde la red host-only). Confirmarlo con `sudo firewall-cmd --list-services` y `sudo firewall-cmd --zone=internal --list-services` antes y después de `romper.sh`.
- **Probar `semanage port -m -t syslogd_port_t -p tcp 514`** antes del Lab 2.2 y comprobar con `sudo ss -tlnp | grep :514` que rsyslog enlaza. Anotar si en esta VM hacía falta o no, para decirlo en clase. Se revierte con `sudo semanage port -d -p tcp 514`.
- Tener los tres scripts y los ocho tickets en un archivo de texto para pegarlos en el chat por bloques.
- Si se usará la VM del instructor como colector rsyslog para todos (Lab 2.2), abrir 514/tcp y anotar su IP host-only; los participantes cambian `192.168.56.10` por esa IP en `cliente.conf`. Comprobar antes que dos VMs en host-only se ven (`ping`).
- Snapshot `dia10-inicio` en la VM de referencia y recordar a todos `dia10-pre-romper` antes del reto.
- Preparar un **marcador visible** (compartir una hoja con tickets y minutos) para controlar el tiempo del reto: 25 min para T1–T3, 20 para T4–T5, el resto bonus.
- `dnf install -y sysstat strace lsof` en la VM de referencia y `systemctl enable --now sysstat sysstat-collect.timer` **la noche anterior**, para que `sar -u` ya tenga histórico que enseñar (recién instalado responde `Cannot open /var/log/sa/saNN`).
- Lista de lo que hay que **verificar en la VM antes de la clase** porque depende de la versión o del hipervisor: (1) si `rd.break` da shell directo o pide contraseña; (2) si rsyslog necesita el `semanage port -m` para 514/tcp; (3) el nombre real de la interfaz host-only en UTM (`nmcli device`); (4) el contenido de `/boot/efi/EFI/redhat/grub.cfg`; (5) cuánto tarda el relabel de SELinux; (6) si `podman generate systemd` sigue existiendo en la versión instalada (solo se menciona).

### Qué estudiar si es nuevo en RHEL (la noche anterior, en la VM)
1. **`rd.break` y emergency mode**: `man dracut.cmdline` (sección `rd.break`), `man systemd.special` (`rescue.target`, `emergency.target`), `man bootup`. Practicar el ciclo completo dos veces, con y sin relabel, hasta hacerlo en 3 minutos. Es la parte del examen que más nervios da y la que más impresiona en clase si sale limpia.
2. **GRUB2 en RHEL 9 con BLS**: `man grubby`, `/usr/share/doc/grub2-common/`, `man grub2-editenv`. Entender por qué `grub2-mkconfig` no toca las entradas de kernel (BLS) y por qué `grubby --args` sí es persistente. Ver `ls /boot/loader/entries/` y `cat` de una entrada.
3. **journalctl por campos**: `man journalctl`, `man systemd.journal-fields`. Practicar `-o verbose` sobre 3 mensajes distintos y filtrar por `_COMM`, `_UID`, `SYSLOG_IDENTIFIER`, `_TRANSPORT`. `man journald.conf` para `Storage`, `SystemMaxUse`, `MaxRetentionSec`.
4. **rsyslog 8 (RainerScript)**: `man rsyslog.conf`, `/usr/share/doc/rsyslog/html/` (si está instalado `rsyslog-doc`). Montar el servidor+cliente en la misma VM **sin** el ruleset para ver el bucle (el log crece sin parar; `systemctl stop rsyslog` lo corta) y luego con el ruleset. Practicar `rsyslogd -N1` con un error a propósito.
5. **auditd**: `man auditctl`, `man ausearch`, `man aureport`, `man augenrules`. Crear la regla, disparar `useradd`, buscar por `-k`, por `-ua` y con `-i`. Ver `/var/log/audit/audit.log` en crudo para entender por qué no se usa `grep`.

### Errores frecuentes de los participantes y cómo resolverlos

| Síntoma | Causa | Solución |
|---|---|---|
| `journalctl --list-boots` solo muestra `0` | Journal no persistente (`/var/log/journal` no existe) | `sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald`; o `Storage=persistent` |
| `journalctl -u httpd` vacío "pero el servicio corre" | El usuario no puede leer el journal del sistema. `student` sí puede porque está en `wheel` (systemd pone ACL para `wheel` y `adm` sobre `/var/log/journal`); cualquier otro usuario, no | `sudo journalctl -u httpd`, o añadir al usuario a `adm`/`systemd-journal` (`sudo usermod -aG systemd-journal usuario`) y volver a iniciar sesión |
| `ss` no muestra 514/tcp tras configurar `imtcp` | SELinux: 514/tcp es `rsh_port_t`, no `syslogd_port_t` | `sudo ausearch -m AVC -ts recent \| grep rsyslogd`; `sudo semanage port -m -t syslogd_port_t -p tcp 514`; `sudo systemctl restart rsyslog` |
| rsyslog no arranca tras editar | Error de sintaxis (comilla, paréntesis) | `sudo rsyslogd -N1` muestra línea y archivo; `journalctl -u rsyslog -n 5` |
| `/var/log/remoto.log` crece sin parar y el disco se llena | Se omitió el `ruleset` en `remoto.conf`: bucle cliente→servidor→cliente | `sudo systemctl stop rsyslog`, corregir, `: > /var/log/remoto.log`, `start` |
| `remoto.log` no recibe nada | Firewall sin 514/tcp, o `cliente.conf` apunta a otra IP, o SELinux (`name_bind`) | `firewall-cmd --list-ports`; `ss -tnp \| grep 514`; `ausearch -m AVC \| grep rsyslogd` → `semanage port -m -t syslogd_port_t -p tcp 514` |
| `logger -n` no muestra error pero no llega | `logger` usa UDP por defecto; el servidor solo escucha TCP | Añadir `-T` (TCP) o cargar `imudp` en el servidor |
| `logrotate -f` → `error: destination ... already exists` | Dos rotaciones el mismo día con `dateext` (ocurre si se invoca `/etc/logrotate.conf` en vez del fragmento) | Borrar el archivo rotado, o añadir `dateformat -%Y%m%d-%H%M%S` al bloque |
| Los rotados salen `.1` y `.2.gz` y no con la fecha | Al invocar `logrotate <fragmento>` no se aplican los globales de `/etc/logrotate.conf` (`dateext`) | Es lo esperado en el lab; con el timer diario el nombre sí llevará la fecha |
| `logrotate -d` dice `log does not need rotating` | Normal: el estado dice que ya rotó hoy | `-f` para forzar; ver `/var/lib/logrotate/logrotate.status` |
| `augenrules --load` → `No change` y `auditctl -l` sin las reglas | El archivo no termina en `.rules` o está en `/etc/audit/` en vez de `rules.d/` | Revisar ruta y extensión; `sudo augenrules --check` |
| `auditctl: Error sending ... Operation not permitted` | Reglas inmutables (`-e 2`) | Solo se puede cambiar reiniciando |
| El menú de GRUB no aparece / no da tiempo a pulsar `e` | `GRUB_TIMEOUT` corto o `menu_auto_hide` | Lab 3.1; en UTM pulsar `Esc` repetidamente al arrancar |
| En GRUB, `e` pide usuario y contraseña | Contraseña de GRUB configurada | `grub2-setpassword` la puso; en la VM del curso no debería estar: revisar `/boot/grub2/user.cfg` |
| Tras `rd.break`, prompt pide contraseña de root | Comportamiento de algunas 9.x | Alternativa `init=/bin/bash` (paso 7 del Lab 3.2) |
| `passwd` en `rd.break` → `Authentication token manipulation error` | Falta `mount -o remount,rw /sysroot` o no se hizo `chroot` | Remontar, `chroot /sysroot`, repetir |
| Tras el cambio de contraseña, nadie puede iniciar sesión | Faltó `/.autorelabel`: `/etc/shadow` con `unlabeled_t` | Volver a `rd.break`, `touch /sysroot/.autorelabel` (o dentro del chroot `touch /.autorelabel`), salir |
| El relabel "se queda colgado" | Es lento (minutos), sobre todo en UTM emulado | Esperar; el contador de `*` avanza; reinicia solo |
| Emergency mode: `vi /etc/fstab` → `Read-only file system` | `/` montado ro | `mount -o remount,rw /` |
| Salgo de emergency con `exit` y vuelve a caer | No se corrigió fstab o falta `daemon-reload` | Corregir línea, `systemctl daemon-reload`, `mount -a` sin errores, luego `systemctl default` |
| `systemctl isolate rescue.target` por SSH y se pierde la sesión | rescue no tiene red ni sshd | Solo desde la consola; volver con `systemctl default` |
| Reto T4: `su - dev02` falla incluso desbloqueado | faillock por intentos previos, o shell `nologin` | `faillock --user dev02 --reset`; `usermod -s /bin/bash dev02` |
| Reto T7: `systemctl --user` → `Failed to connect to bus` | Se entró con `sudo su - student` | Salir y entrar por SSH directo como `student` |
| Reto T3: `rm .cache_old.img` y `df` sigue en 100 % | Un proceso lo tenía abierto (si alguien hizo `tail -f` o `less`) | `sudo lsof +L1`; cerrar el proceso |
| `verificar-web01.sh` T3 falla aunque hay espacio | Falta `/usr/local/bin/backup.sh` o `/home/student/empresa` | Volver a ejecutar `construir-web01.sh` (idempotente) |

### Diferencias VirtualBox (x86_64) vs UTM (aarch64)
- **GRUB/UEFI:** UTM arranca siempre por UEFI: existe `/boot/efi`, `grub2-mkconfig` produce `Adding boot menu entry for UEFI Firmware Settings`, y el editor de GRUB puede mostrar la línea como `linux` con `/vmlinuz-...aarch64`. VirtualBox por defecto usa BIOS (sin `/boot/efi`); si el participante activó EFI en la VM, se comporta como UTM. En ambos casos el destino de `grub2-mkconfig` en RHEL 9 es `/boot/grub2/grub.cfg`.
- **Ver el menú:** en UTM conviene pulsar `Esc` justo al aparecer el logo; en VirtualBox basta con mover una flecha durante los 10 segundos.
- **Relabel y arranque:** en UTM emulado (no nativo) el relabel y el arranque son notablemente más lentos; el instructor debe advertir que su VM tarda más que las de los participantes.
- **Discos:** `vda` (UTM) vs `sda` (VirtualBox) en `iostat`, `lsblk`, `dmesg`; el LV `/dev/mapper/vg_datos-lv_backups` se llama igual en ambos.
- **Interfaces y red host-only:** `enp0s1/enp0s2` (UTM, ⚠️ **verificar con `nmcli device`**) vs `enp0s3/enp0s8` (VirtualBox); el perfil se llama `lab` en ambos; la red host-only de UTM puede no ser `192.168.56.0/24` (cada uno usa su IP en `cliente.conf`, `exports`, el origen de la zona `internal` y el navegador). Si la red host-only del instructor es otra (por ejemplo `192.168.64.0/24`), el origen de la zona `internal` del Día 8 y el `/etc/exports` deben llevar **esa** red, no la del material.
- **Consola en UTM:** el port forwarding NAT existe solo con "Emulated VLAN"; para el navegador usar la IP host-only. La consola de UTM no captura `Ctrl+X` de forma distinta, pero `Ctrl+E` para ir al final de la línea de GRUB puede no funcionar en algunos teclados: usar la tecla `Fin` o las flechas.
- **Kernel:** `rpm -q kernel` termina en `.aarch64`; `grubby --info` muestra `/boot/vmlinuz-...aarch64`. `crashkernel=` puede tener un valor distinto.

### Preguntas probables y respuesta corta
- **¿Si cualquiera con la consola puede cambiar la contraseña de root, para qué sirve la contraseña?** Para el acceso remoto. El acceso físico siempre ha equivalido a root; se mitiga con contraseña en GRUB (`grub2-setpassword`), cifrado de disco (LUKS) y control físico del centro de datos. En una VM, con control del hipervisor.
- **¿Por qué el relabel? ¿No basta con cambiar la contraseña?** Porque `passwd` en `rd.break` corre sin política SELinux y el `/etc/shadow` nuevo nace sin etiqueta; con la política cargada nadie podría leerlo. El relabel lo corrige (o `load_policy -i` + `restorecon`, si se domina).
- **¿`nofail` en todo?** En discos de datos sí; en `/`, `/boot`, `/var` no: si faltan, es mejor que el servidor se detenga y avise. Y para NFS, `_netdev,nofail`.
- **¿Journald reemplaza a rsyslog? ¿Puedo apagar uno?** journald es obligatorio (parte de systemd). rsyslog se puede apagar si nadie lee `/var/log/messages` ni hay colector, pero casi toda institución lo necesita para centralizar; lo habitual es mantener ambos y limitar el journal.
- **¿Los logs remotos por TCP son seguros?** Van en claro. Para producción: `imtcp` con TLS (`rsyslog-gnutls`, puerto 6514) o `omrelp`. En una red de administración aislada, TCP claro es aceptable a corto plazo.
- **¿Qué diferencia hay entre auditd y los logs normales?** auditd registra en el kernel, antes de que la aplicación pueda "olvidarse" de escribir; es a prueba de manipulación por el propio proceso vigilado y es lo que exigen normas (PCI, ISO 27001) para trazabilidad de accesos a archivos sensibles.
- **¿`dnf update` puede dejarme sin arrancar?** Rara vez, y por eso conserva el kernel anterior: en GRUB se elige el viejo y se investiga. Con snapshot antes de actualizar, el riesgo es cero.
- **¿Cómo sé si el servidor se reinició solo o alguien lo apagó?** `last -x | grep -E 'shutdown|reboot'` y `journalctl -b -1 -n 20`: un apagado limpio deja `Reached target Power-Off`; un corte no deja nada.
- **¿Cockpit sustituye a todo esto?** No; es una vista amigable sobre los mismos comandos (usa `journalctl`, `systemctl`, `nmcli` por debajo). Sirve para técnicos de guardia y para mostrar el estado a un jefe; el diagnóstico fino sigue siendo terminal.
- **¿Y si el examen me pide algo que no vimos?** Las páginas `man` y `/usr/share/doc` están disponibles en el examen. La habilidad que cuenta es saber buscar: `man -k palabra`, `apropos`, `man 5 archivo`.

### Relación con el examen RHCSA (EX200, RHEL 9)
- **Operate running systems:** "Boot systems into different targets manually" (`systemctl isolate`, `systemd.unit=` en GRUB); "**Interrupt the boot process in order to gain access to a system**" (`rd.break`, Lab 3.2); "Identify CPU/memory intensive processes and kill processes" (`top`, `ps --sort`, Bloque 1); "Locate and interpret system log files and journals" (Bloque 2); "**Preserve system journals**" (`Storage=persistent`, `/var/log/journal`); "Start, stop, and check the status of network services"; "Securely transfer files between systems" (Día 5).
- **Deploy, configure, and maintain systems:** "**Configure systems to boot into a specific target automatically**" (`set-default`); "**Modify the system bootloader**" (`grubby`, `/etc/default/grub`, `grub2-mkconfig`); "Schedule tasks using at and cron" (T3 vuelve a exigirlo).
- **Create and configure file systems / Configure local storage:** el ticket T6 y el Lab 3.3 son el objetivo implícito "diagnose and correct boot issues from fstab"; T3 practica "Diagnose and correct file permission problems" y `df`/`du`.
- **Manage security:** T5 ("restore default file contexts"), T1 ("configure firewall settings"), auditd/`ausearch` ("diagnose and address routine SELinux policy violations").
- **Manage containers:** T7 ("configure a container to start automatically as a systemd service").
- **Fuera del EX200 pero en la ficha del cliente:** rsyslog remoto, plantillas, logrotate a fondo, auditd con reglas, `sysstat`, `strace`. Se incluyen porque son trabajo diario en un centro de datos institucional.

---

## Tarea y preparación para el día siguiente

No hay día siguiente, pero sí un plan de dos semanas:

1. **Snapshot `dia10-fin`** con la VM apagada, después de `verificar-web01.sh` → `Fallas: 0`. Conservar también `dia10-pre-romper`.
2. **Repetir el reto mañana**: restaurar `dia10-pre-romper`, ejecutar `romper.sh` (con T6 esta vez si hoy se usó `--sin-fstab`) y resolver los ocho tickets **cronometrando**. Meta: T1–T3 en menos de 20 minutos; los ocho en menos de 60.
3. **Práctica de 20 min** durante 10 días, sin mirar el material:
   - Día a día, un lab de cada sesión desde `dia01-fin` (usuarios y permisos, LVM con `lvextend -r`, `nmcli` con IP estática, `semanage port` + `fcontext`, Quadlet con linger, autofs, cron y timer).
   - Cada 3 días, `rd.break` completo (con relabel) y un `fstab` roto, hasta que salgan sin pensar.
   - Antes de cada `reboot`: `findmnt --verify`, `mount -a`, `systemctl --failed`, `apachectl configtest`, `rsyslogd -N1`.
4. Lecturas cortas: `man journalctl` (sección "FILTERING OPTIONS"), `man grubby` (EXAMPLES), `man 5 logrotate.conf` (opciones `daily`, `rotate`, `postrotate`), `man ausearch` (`-k`, `-ts`, `-i`), objetivos oficiales del EX200 en redhat.com (marcar en la tabla del cierre los que aún cuestan).
5. Enviar al instructor, en una semana, la tabla de los 8 tickets con las 4 líneas cada uno (qué, cómo lo encontré, cómo lo corregí, cómo lo verifiqué) del segundo intento cronometrado. Es la última entrega del curso y la mejor medida de lo aprendido.
