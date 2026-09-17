# 4 — `web01`: qué tiene y cómo se verifica (guía del instructor)

Cinco minutos. Una sola idea: **todos van a tener el mismo servidor**, construido a mano en el lab que sigue, y **verificado desde donde lo ve el usuario** antes de romperlo. Tipeás los tres comandos del bloque bash mientras señalás la tabla; el diagrama de las dos zonas se lee en voz alta.

**Qué es `web01`:** el nombre del servidor del reto. Hasta hoy la VM se llamaba `rhel01`; en el lab se renombra para que el escenario se sienta como otro servidor.
**Qué es `hostnamectl set-hostname`:** cambiar el nombre del servidor de forma permanente (Día 5). `hostnamectl --static` muestra el guardado.
**Qué es FQDN:** *Fully Qualified Domain Name*: el nombre completo con dominio, `web01.lab.local`. `hostname` a secas también lo muestra completo si así se guardó.
**Qué es `/srv/compartido`:** la carpeta del área, igual que `/srv/sistemas` del Día 3: dueño `root`, grupo `sistemas`, `2770` (el `2` es setgid: lo que se cree adentro hereda el grupo).
**Qué es `dev01` / `dev02`:** dos desarrolladores nuevos, miembros de `sistemas`. Los tickets T2 y T4 son sobre ellos.
**Qué es `/var/www/html`:** la carpeta que Apache sirve de fábrica (Día 9). Su contexto SELinux es `httpd_sys_content_t`, y todo lo que se cree adentro lo hereda: por eso hoy el sitio va ahí y no en `/web` como el Día 8.
**Qué es `00-default.conf`:** el sitio por defecto de Apache que armaron el Día 9, con `DocumentRoot /var/www/html`. `grep DocumentRoot` en ese archivo confirma que sigue apuntando ahí.
**Qué es `httpd_sys_content_t` / `admin_home_t`:** el contexto que Apache puede leer / el contexto de lo que se crea en `/root`. Un archivo movido con `mv` desde `/root` arrastra `admin_home_t` y Apache responde 403 (Día 8). Es el ticket T5.
**Qué es `curl -s http://localhost/`:** pedir la página desde la propia VM, sin barra de progreso (`-s`). Prueba que Apache sirve; **no** prueba el firewall, porque `localhost` no pasa por él.
**Qué es `ALREADY_ENABLED`:** el aviso de `firewall-cmd` cuando lo que se agrega ya estaba. No es error.
**Qué es NAT / host-only / 8080:** las dos redes de la VM (Día 5). NAT sale a internet y tiene el port forwarding `8080→80` y `2222→22`; host-only es la red directa entre la computadora y la VM (`192.168.56.10`). El NAT entra por la zona `public`; la host-only por `internal` (Día 8).
**Qué es `/etc/cron.d/`:** las tareas programadas **del sistema**, un archivo por tarea, con el usuario antes del comando (Día 7). Distinto del `crontab -e` de cada usuario, que no lleva usuario.
**Qué es `/usr/local/bin`:** la carpeta para scripts del sistema que corren root, cron o systemd (Día 7). `~/bin` es para scripts personales; root no debe ejecutar scripts que un usuario puede editar.
**Qué es `backup.sh`:** el script del Día 7 (`backup.sh ORIGEN DESTINO`): hace un `tar.gz` del origen en el destino, borra los de más de 7 días, escribe en el log y sale con `0` si todo bien, `2` si `tar` falló. Hoy se copia a `/usr/local/bin` para que lo corra root.
**Qué es `lv_backups` / `/backups`:** el volumen lógico de 1.5 GB en ext4 que crearon en el reto del Día 6, montado en `/backups` con `nofail`. Tener los respaldos en su propio volumen es lo que permite que el ticket T3 llene **ese** disco y no la raíz.
**Qué es `findmnt /backups`:** muestra qué dispositivo está montado ahí. Si no responde nada, `/backups` es una carpeta común de la raíz, no un montaje (Día 6).
**Qué es `dia10-pre-romper`:** el snapshot de la VM sana, justo antes de ejecutar `romper.sh`. Restaurarlo no resuelve ningún ticket: sirve para empezar el intento de nuevo.

---

## La tabla de piezas

**Qué decir:** "cada fila es algo que armaron en un día del curso. El reto rompe cinco de estas piezas; la columna de la derecha es cómo se verifica cada una, y es exactamente el bloque de verificación del reto."
**Qué señalar:** la fila de Apache dice **activo y habilitado**: `start` sin `enable` es el ticket que vuelve mañana.

---

## Las dos zonas

Leer el diagrama en voz alta, las dos líneas. **La frase:** "es el error número uno del curso: abren `http` en `public`, prueban desde `localhost:8080`, funciona, y la oficina sigue sin web porque la oficina entra por `internal`. Todo lo que se publica va en las dos zonas."

---

## Verificar desde donde lo ve el usuario

**Qué decir:** "`curl localhost` no es verificación: es una pista. La verificación es el navegador de su computadora, por las dos direcciones. Y para T2, `sudo -u dev01 touch`: si dev01 no puede escribir, el ticket sigue abierto aunque `ls -ld` se vea lindo."
