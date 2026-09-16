# Curso Básico de Linux con Red Hat Enterprise Linux (RHEL)

**Duración:** 40 horas · 10 sesiones de 4 h
**Modalidad:** virtual, en vivo, con pantalla compartida del instructor
**Nivel:** principiante, sin conocimientos previos de Linux
**Enfoque:** práctico; cada sesión combina explicación breve, práctica guiada y un reto individual

---

## Objetivo del curso

Al finalizar, el participante podrá entrar a un servidor Linux, orientarse en él, identificar qué está ocurriendo y resolver los problemas más comunes de administración básica. El curso es deliberadamente acotado: construye una base práctica y funcional, no forma administradores avanzados en 40 horas.

### Al terminar, el participante será capaz de
- Instalar RHEL en una máquina virtual y acceder por SSH.
- Navegar el sistema de archivos, crear, mover y editar archivos desde la terminal.
- Leer y filtrar logs para investigar un problema.
- Administrar usuarios, grupos y permisos, y usar `sudo` correctamente.
- Instalar software con `dnf` y administrar procesos.
- Iniciar, detener y habilitar servicios con `systemctl`, y publicarlos a través del firewall.
- Configurar acceso remoto por SSH y transferir archivos.
- Agregar un disco al servidor y montarlo de forma persistente.
- Automatizar un respaldo con un script y `cron`.
- Diagnosticar de forma ordenada un servidor con fallas.

---

## Requisitos previos y logística

### Requisitos del participante
- Computadora con al menos 8 GB de RAM y 40 GB de disco libres.
- Windows o Mac Intel: VirtualBox. Mac con chip Apple (M1/M2/M3): UTM o VMware Fusion, con la ISO de RHEL para aarch64.
- Cuenta gratuita en developers.redhat.com (suscripción de desarrollador) e ISO de RHEL 9 descargada **antes de la sesión 1**.
- Cámara y micrófono.

Se enviará a los participantes una guía de preparación con estos pasos una semana antes del inicio.

### Configuración estándar de la VM
- 2 vCPU, 3 GB RAM, 20 GB de disco, red NAT.
- Instalación tipo "Server" (sin interfaz gráfica).
- Port forwarding NAT: puerto 2222 del host → 22 de la VM. A partir de la sesión 2, los participantes acceden con `ssh -p 2222 student@localhost` desde la terminal de su propio equipo.
- Se pondrá a disposición una imagen (OVA) preinstalada como respaldo para quien tenga problemas con la instalación.

### Nota sobre el equipo del instructor
El instructor demuestra desde una VM RHEL para aarch64 (Mac Apple Silicon). Los comandos son idénticos; las únicas diferencias visibles son la arquitectura reportada (`aarch64` vs. `x86_64`) y el nombre de los discos (`/dev/vda` vs. `/dev/sda`), que se señalarán explícitamente en la sesión 8.

---

## Estructura de cada sesión (240 min)

| Bloque | Min | Descripción |
|---|---:|---|
| Repaso | 10 | Tres preguntas sobre la sesión anterior, con la terminal abierta |
| Bloque 1 | 55 | 15–20 min de concepto + 35 min de práctica guiada |
| Bloque 2 | 55 | Igual |
| Descanso | 15 | |
| Bloque 3 | 50 | Igual |
| Reto individual | 40 | El participante resuelve solo; el instructor asiste |
| Cierre | 15 | Resumen, entrega del cheatsheet de la sesión y snapshot de la VM |

Durante la práctica guiada, cada comando que el instructor ejecuta lo ejecutan los participantes en su propia VM. Cada 15–20 minutos se solicita un *checkpoint*: los participantes pegan en el chat la salida de un comando para verificar que todos van al mismo ritmo.

---

## Sesión 1 — Instalar Linux y entrar

**Meta:** todos terminan con una VM RHEL funcionando y saben acceder a ella.

**Conceptos (máx. 30 min en total):** qué es Linux, kernel vs. distribución, qué es RHEL y por qué se usa en empresas, qué es una máquina virtual, qué es una ISO.

**Práctica guiada:**
1. Crear la VM (CPU, RAM, disco, red).
2. Montar la ISO y arrancar.
3. Instalador: idioma, teclado, zona horaria, particionado automático, usuario `student` con privilegios de administrador, hostname `rhel01`, red activada.
4. Reiniciar e iniciar sesión.
5. Primeros comandos:

```bash
whoami
hostname
cat /etc/redhat-release
ip addr
ping -c 3 google.com
```

6. Configurar port forwarding y acceder por SSH desde el equipo propio (como receta; se explica en la sesión 7).
7. Crear snapshot de la VM.

**Si sobra tiempo:** `man`, `--help`, autocompletado con Tab, historial con flechas, `clear`, `exit`.

**Reto:** ninguno; el reto de la sesión es la instalación.

---

## Sesión 2 — Terminal I: moverse y manejar archivos

**Meta:** navegar el sistema sin interfaz gráfica y manejar archivos y carpetas.

**Temas:** qué es la shell y el prompt; estructura de un comando; rutas absolutas y relativas, `.`, `..`, `~`; el árbol de Linux (`/home`, `/etc`, `/var/log`, `/tmp`); `man`, `--help`, Tab, historial.

**Comandos:** `pwd cd ls mkdir touch cp mv rm rmdir`, `ls -l`, `ls -a`, `rm -r`, `ls -R`

**Lab guiado — Estructura de la empresa**

```text
~/empresa/
├── documentos/
├── clientes/
├── backups/
└── logs/
```

1. Crear la estructura con `mkdir -p`.
2. Crear cinco archivos con `touch` en `documentos/`.
3. Copiar `documentos/` completo a `backups/` (`cp -r`).
4. Mover un archivo de `documentos/` a `clientes/`.
5. Renombrar un archivo con `mv`.
6. Borrar un archivo y luego una carpeta completa.
7. Verificar con `ls -l` y `ls -R`.

**Reto individual:** reproducir exactamente una estructura de directorios distinta entregada por el instructor y enviar la salida de `ls -R ~/empresa`.

---

## Sesión 3 — Terminal II: leer, buscar y editar

**Meta:** leer y filtrar archivos de texto, y editarlos sin interfaz gráfica.

**Temas:** `cat`, `less`, `head`, `tail`, `tail -f`; `grep` (`-i`, `-n`, `-r`); `find /ruta -name`; redirección `>` y `>>`; pipe `|` con `grep`, `wc -l`, `sort`; editor **nano** y **vi básico** (`i`, `Esc`, `:wq`, `:q!`).

**Lab guiado — Logs de un servidor**

```bash
mkdir -p ~/empresa/logs
echo "INFO servidor iniciado" > ~/empresa/logs/server.log
echo "ERROR conexion a base de datos fallida" >> ~/empresa/logs/server.log
echo "INFO usuario ana conectado" >> ~/empresa/logs/server.log
echo "ERROR disco lleno" >> ~/empresa/logs/server.log
```

1. Ver el archivo con `cat`, `less`, `head -2`, `tail -1`.
2. Filtrar los `ERROR` con `grep`.
3. Contar errores: `grep ERROR server.log | wc -l`.
4. Guardar los errores en `errores.txt` con redirección.
5. Abrir `errores.txt` con nano, agregar una línea, guardar. Repetir con vi.
6. `find ~/empresa -name "*.log"`.
7. Un log real: `sudo tail -20 /var/log/messages`, `sudo grep -i error /var/log/messages`.

**Reto individual:** con un archivo de 200 líneas provisto por el instructor, indicar cuántas líneas contienen `FAIL`, cuál es la última línea con `WARN`, y crear un archivo solo con las líneas del usuario `ana`.

---

## Sesión 4 — Usuarios, grupos, permisos y sudo

**Meta:** entender quién puede hacer qué en el sistema.

**Temas:** `root` vs. usuario normal; `sudo` y por qué no se trabaja como root; usuarios y grupos; propietario y grupo de un archivo; permisos `r w x` y la salida de `ls -l`; notación numérica (`755`, `644`, `770`); bit setgid (`2770`) en carpetas compartidas.

**Comandos:** `id whoami groups sudo useradd usermod -aG userdel groupadd passwd chmod chown chgrp su -`

**Lab guiado — Empresa PanamaTech**

```text
Usuarios: ana, carlos (grupo developers)   pedro (grupo support)
Carpetas: /srv/development  /srv/support
```

1. Crear grupos y usuarios, asignar contraseñas.
2. Agregar usuarios a grupos; verificar con `id`.
3. Crear carpetas, `chown root:developers /srv/development`, `chmod 2770 /srv/development`.
4. Como `ana`, crear un archivo dentro y comprobar con `ls -l` que hereda el grupo.
5. Como `pedro`, intentar entrar a `/srv/development` y observar "Permission denied".
6. Como `pedro`, intentar `sudo cat /etc/shadow` y observar el mensaje de sudoers.

**Reto individual:** configurar `/srv/support` para que cualquier usuario pueda leer su contenido pero solo el grupo `support` pueda escribir, y demostrarlo con `ana` (lectura correcta, escritura denegada).

---

## Sesión 5 — Paquetes y procesos

**Meta:** instalar software y ver o finalizar lo que se está ejecutando.

**Temas:** paquetes y repositorios; `dnf search/info/install/remove/update`; `rpm -qa`; procesos y PID; `ps aux`, `top`, `kill`, `kill -9`; `&` y `Ctrl+C`.

**Lab guiado**
1. `dnf search htop`, `dnf info htop`, `sudo dnf install htop`; verificar con `rpm -qa | grep htop`; eliminarlo.
2. Instalar `tree`, `vim`, `wget`.
3. `sudo dnf update -y`.
4. Lanzar `sleep 600 &`; encontrarlo con `ps aux | grep sleep`; finalizarlo con `kill`. Repetir con `top`.
5. Lanzar `yes > /dev/null &`; observarlo en `top`; finalizarlo.

**Reto individual:** un proceso desconocido consume la CPU del servidor. Identificar su PID y usuario, y finalizarlo.

---

## Sesión 6 — Servicios, systemd, logs y servidor web

**Meta:** instalar Apache, administrarlo como servicio, publicarlo y leer sus logs.

**Temas:** qué es un servicio; `systemctl status/start/stop/restart/enable/disable`; *enabled* vs. *active*; `journalctl -u`, `/var/log/`; `firewalld`: `--list-all`, `--add-service`, `--permanent`, `--reload`.

**Lab guiado**
```bash
sudo dnf install httpd -y
systemctl status httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<h1>Servidor rhel01</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```
1. Comprobar que funciona localmente.
2. Intentar desde el navegador del equipo propio (port forwarding 8080 → 80): no carga.
3. Permitir HTTP en el firewall y recargar. Ahora carga.
4. Detener Apache, observar `status` y `journalctl -u httpd`.
5. `sudo tail -f /var/log/httpd/access_log` mientras se recarga la página.
6. Reiniciar la VM y comprobar que Apache inicia solo.

**Reto individual:** el instructor deja la web sin funcionar (servicio detenido o firewall cerrado). Restaurarla y explicar qué estaba mal.

---

## Sesión 7 — Redes básicas y SSH

**Meta:** identificar la configuración de red propia y administrar otro servidor remotamente.

**Temas:** IP, máscara, gateway, DNS (conceptual); `ip addr`, `ip route`, `/etc/resolv.conf`, `ping`, `ss -tulpn`, `nmcli device`; SSH; `ssh-keygen` y `ssh-copy-id`; `scp`.

**Lab guiado — Administrar el servidor remotamente**
El equipo del participante actúa como cliente y su VM como servidor (la conexión por SSH ya funciona desde la sesión 1). El instructor demuestra en pantalla la conexión entre dos VMs para ilustrar el concepto; los participantes no necesitan montar una segunda VM.

1. En la VM: `ip addr`, `ip route`, `cat /etc/resolv.conf`; `ping` a un host externo.
2. `ss -tulpn`: identificar SSH (22) y Apache (80) escuchando.
3. Desde el equipo propio: `ssh -p 2222 student@localhost`; confirmar con `hostname`; `exit`.
4. Crear un archivo en el equipo propio y transferirlo con `scp -P 2222 archivo.txt student@localhost:/tmp/`; verificar en la VM.
5. En el equipo propio: `ssh-keygen`, `ssh-copy-id -p 2222 student@localhost`; volver a entrar sin contraseña.
6. Explicar qué es el port forwarding usado desde la sesión 1 y qué cambia cuando el servidor está en otra máquina real.

**Reto individual:** desde el equipo propio, instalar `tree` en la VM sin abrir sesión interactiva (`ssh -p 2222 student@localhost "sudo dnf install -y tree"`) y copiar `/etc/hostname` de la VM al equipo propio con `scp`.

---

## Sesión 8 — Discos y almacenamiento

**Meta:** agregar un disco al servidor y saber cuánto espacio hay.

**Temas:** disco vs. filesystem vs. punto de montaje; dispositivos en `/dev` (`sdb` en VirtualBox, `vdb` en UTM/Fusion); `lsblk`, `df -h`, `du -sh`; `mkfs.xfs`; `mount`/`umount`; `/etc/fstab` con UUID (`blkid`); `mount -a` antes de reiniciar. LVM y particiones: solo mención conceptual.

**Lab guiado**
1. Apagar la VM, agregar un disco de 5 GB, encender.
2. `lsblk` para identificarlo.
3. `sudo mkfs.xfs` sobre el nuevo disco.
4. `sudo mkdir /data`, montar, `df -h`.
5. Crear un archivo, desmontar, montar de nuevo.
6. `blkid`, agregar la línea a `/etc/fstab`: `UUID=... /data xfs defaults,nofail 0 0` (`nofail` evita que un error en la línea impida el arranque).
7. `sudo umount /data && sudo mount -a && df -h`; reiniciar y verificar.
8. `du -sh /var/log/* | sort -h`.

**Reto individual:** `/data` aparece lleno sin motivo aparente. Encontrar la causa y liberar espacio.

---

## Sesión 9 — Bash básico, tar y cron

**Meta:** automatizar una tarea repetitiva.

**Temas:** qué es un script, shebang, `chmod +x`, variables, `$(date +%F)`, `if` de existencia de directorio, `tar -czf` / `-xzf` / `-tzf`, `crontab -e`. Introducción a SELinux: `getenforce`, `ls -Z`, `restorecon`.

**Lab guiado — backup.sh**
```bash
#!/bin/bash
FECHA=$(date +%F)
ORIGEN=/home/student/empresa
DESTINO=/data/backups

if [ ! -d "$DESTINO" ]; then
    mkdir -p "$DESTINO"
fi

tar -czf "$DESTINO/empresa-$FECHA.tar.gz" "$ORIGEN"
echo "Backup completado: $DESTINO/empresa-$FECHA.tar.gz"
```
1. Escribir el script, dar permisos, ejecutar, verificar con `ls -lh /data/backups`.
2. Listar el contenido con `tar -tzf`; restaurar en `/tmp` con `tar -xzf`.
3. Programarlo con `crontab -e`; probar con `* * * * *` durante un minuto.
4. SELinux (20 min): crear `index2.html` en `/root`, moverlo a `/var/www/html/`, observar 403 con `curl`, `ls -Z` muestra el contexto incorrecto, `sudo restorecon -v` lo corrige. Idea clave: si un servicio no puede leer un archivo con permisos correctos, revisar SELinux.

**Reto individual:** modificar el script para recibir el origen como argumento (`$1`) y borrar backups de más de 7 días (`find ... -mtime +7 -delete`).

---

## Sesión 10 — Troubleshooting y proyecto final

**Meta:** demostrar capacidad de diagnóstico sin indicaciones.

**Bloque 1 (45 min) — Método.** Antes de tocar nada: ¿qué debería estar pasando? ¿qué está pasando? Orden de revisión: **servicio → logs → red/firewall → permisos → disco → SELinux**.

**Bloque 2 (30 min) — Construcción.** Dejar listo `web01`: hostname, grupo `developers` con `dev01` y `dev02`, `/srv/company` compartida, Apache habilitado y publicado, `/data/backups` con el script de respaldo.

**Bloque 3 (90 min) — Servidor con fallas.** El instructor introduce cinco fallas en la VM de cada participante (Anexo A). El participante recibe tickets, no pistas:

```text
Obligatorios:
Ticket 1: "La página web de la empresa no carga desde afuera."
Ticket 2: "dev01 dice que no puede guardar archivos en /srv/company."
Ticket 3: "El backup de anoche falló por falta de espacio."

Bonus:
Ticket 4: "dev02 no puede iniciar sesión."
Ticket 5: "La web muestra un error 403 aunque el archivo está ahí."
```

**Evaluación:** se evalúan los tres tickets obligatorios; los bonus son opcionales para quien termine antes. Por cada ticket, el participante indica qué estaba mal, cómo lo encontró, cómo lo corrigió y cómo lo verificó. Se considera resuelto solo si la verificación final es correcta (por ejemplo, el ticket 1 requiere que la web cargue desde fuera de la VM, no solo con `curl localhost`).

**Cierre (30 min):** puesta en común del ticket más difícil y recomendaciones para seguir aprendiendo (ruta RHCSA).

---

## Anexo A — Script de fallas para el lab final (uso exclusivo del instructor)

Se ejecuta como root en la VM del participante tras completar el bloque de construcción. El tamaño del `fallocate` debe ajustarse al disco de `/data` dejando unos 500 MB libres.

```bash
#!/bin/bash
# romper.sh - prepara la VM para el lab final. Ejecutar con sudo.

# Ticket 1: servicio detenido y firewall cerrado
systemctl stop httpd
systemctl disable httpd
firewall-cmd --remove-service=http --permanent
firewall-cmd --reload

# Ticket 2: permisos incorrectos en /srv/company
chown root:root /srv/company
chmod 755 /srv/company

# Ticket 3: /data lleno con un archivo oculto
fallocate -l 4G /data/.cache_old.img

# Ticket 4: usuario bloqueado
passwd -l dev02

# Ticket 5: contexto SELinux incorrecto
echo "<h1>Servidor web01 funcionando</h1>" > /root/index.html
mv -f /root/index.html /var/www/html/index.html
```

**Guía de solución**

| Ticket | Diagnóstico esperado | Corrección |
|---|---|---|
| 1 | `systemctl status httpd` inactivo; tras iniciarlo, `curl localhost` funciona pero desde fuera no; `firewall-cmd --list-all` sin http | `systemctl enable --now httpd`; `firewall-cmd --add-service=http --permanent; --reload` |
| 2 | `ls -ld /srv/company` muestra root:root 755 | `chown root:developers`, `chmod 2770` |
| 3 | `df -h` muestra /data lleno; `du -sh /data/*` no lo explica; `ls -la /data` revela el archivo | Borrar `.cache_old.img` |
| 4 | `su - dev02` falla; `passwd -S dev02` indica `L`; `/var/log/secure` | `passwd -u dev02` |
| 5 | `curl` responde 403; permisos correctos; `ls -Z` muestra `admin_home_t`; `journalctl -u httpd` | `restorecon -v /var/www/html/index.html` |

---

## Anexo B — Fallas para los retos de las sesiones 5, 6 y 8 (uso exclusivo del instructor)

```bash
# Sesión 5: proceso que consume CPU con nombre poco evidente
cp /usr/bin/yes /tmp/kworkerd && nohup /tmp/kworkerd > /dev/null 2>&1 &

# Sesión 6 (una de las dos, sin indicar cuál)
systemctl stop httpd
firewall-cmd --remove-service=http --permanent && firewall-cmd --reload

# Sesión 8: disco lleno
fallocate -l 4G /data/.relleno
```