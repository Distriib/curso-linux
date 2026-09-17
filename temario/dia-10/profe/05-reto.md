# Reto — Solución

## Preparar: `romper.sh`

Pegar el script en el chat. Cada uno lo guarda con `vim romper.sh` y ejecuta `sudo bash romper.sh`. Antes, que todos confirmen `findmnt /backups`: la primera línea del script se niega a seguir si `/backups` no es un montaje propio (si no, el `dd` llenaría la raíz).

```bash
#!/bin/bash
# romper.sh - Dia 10. Ejecutar: sudo bash romper.sh
mountpoint /backups || exit 1

# T1: la web deja de verse desde afuera
systemctl disable --now httpd > /dev/null 2>&1
firewall-cmd --permanent --remove-service=http > /dev/null
firewall-cmd --permanent --zone=internal --remove-service=http > /dev/null
firewall-cmd --reload > /dev/null

# T2: la carpeta compartida pierde el grupo y el setgid
chown root:root /srv/compartido
chmod 755 /srv/compartido

# T3: el disco de respaldos se llena con un archivo oculto, y desaparece el cron
dd if=/dev/zero of=/backups/.cache_old.img bs=1M status=none 2> /dev/null
/usr/local/bin/backup.sh /home/student/empresa /backups >> /var/log/backup.log 2>&1
rm -f /etc/cron.d/backup-empresa

# T4: dev02 queda bloqueado y sin shell
passwd -l dev02 > /dev/null
usermod -s /sbin/nologin dev02

# T5: index.html se queda con el contexto de /root
cp /var/www/html/index.html /root/index.html
mv -f /root/index.html /var/www/html/index.html

echo "Listo. El servidor esta averiado. Empiecen por el ticket T1."
```

Decir en voz alta el orden sugerido (T1 → T5 → T2 → T3 → T4) y los tiempos: 25 minutos para T1, T5 y T2; el resto para T3 y T4. Anotar cuánto tarda cada uno por ticket.

## Solución por ticket

| Ticket | Cómo se encuentra | Corrección | Verificación |
|---|---|---|---|
| **T1** | `systemctl status httpd` → `inactive (dead)` y `disabled`. `sudo ss -tlnp` sin el `:80`. Después de arrancarlo, `curl localhost` da 403 (eso es T5) y desde el navegador sigue sin cargar: `sudo firewall-cmd --list-services` y `sudo firewall-cmd --zone=internal --list-services` no tienen `http` | `sudo systemctl enable --now httpd` · `sudo firewall-cmd --permanent --add-service=http` · `sudo firewall-cmd --permanent --zone=internal --add-service=http` · `sudo firewall-cmd --reload` | navegador: `http://192.168.56.10/` **y** `http://localhost:8080/` · `systemctl is-enabled httpd` → `enabled` |
| **T5** | `curl -I http://localhost/` → `403 Forbidden`. `ls -l /var/www/html/index.html` bien (`644`). `ls -Z /var/www/html/index.html` → `admin_home_t`. `sudo ausearch -m AVC -ts recent` muestra a `httpd` denegado sobre `index.html`. `sudo tail -3 /var/log/httpd/error_log` → `Permission denied` | `sudo restorecon -v /var/www/html/index.html` | `ls -Z` → `httpd_sys_content_t` · `curl -s http://localhost/` muestra el `<h1>` · navegador |
| **T2** | `sudo -u dev01 touch /srv/compartido/x` → `Permission denied`. `ls -ld /srv/compartido` → `drwxr-xr-x. root root`. `id dev01` sí está en `sistemas` | `sudo chown root:sistemas /srv/compartido` · `sudo chmod 2770 /srv/compartido` | `sudo -u dev01 touch /srv/compartido/prueba` · `ls -l /srv/compartido` → el archivo con grupo `sistemas` |
| **T3** | `sudo tail -3 /var/log/backup.log` → `ERROR: tar falló al crear ...`. `df -h /backups` → `100%`. `ls /backups` no muestra al culpable; `sudo ls -la /backups` → `.cache_old.img`. `sudo lsof +L1` vacío (no está abierto, no es el caso del Lab 1.1). `ls /etc/cron.d/` → falta `backup-empresa` | `sudo rm /backups/.cache_old.img` · recrear `/etc/cron.d/backup-empresa` con `sudo vim` (las tres líneas del Lab 4.1, Parte 5) | `sudo /usr/local/bin/backup.sh /home/student/empresa /backups` → `OK: creado` · `df -h /backups` bajo · `cat /etc/cron.d/backup-empresa` |
| **T4** | `su - dev02` falla. `sudo passwd -S dev02` → `dev02 LK` (bloqueada). `getent passwd dev02` → termina en `/sbin/nologin`. `sudo faillock --user dev02` (si probó cinco veces, además lo bloqueó faillock, Día 8) | `sudo passwd -u dev02` · `sudo usermod -s /bin/bash dev02` · `sudo faillock --user dev02 --reset` | `su - dev02` con `Pgn.2026` → entra; `exit` · `sudo passwd -S dev02` → `PS` · `getent passwd dev02` → `/bin/bash` |

SSH como `dev02` **no** funciona y no tiene que funcionar: el Día 8 dejó `AllowUsers student`. No es parte del ticket; si alguien lo "arregla", que lo deshaga.

## Verificación

```bash
systemctl is-active httpd; systemctl is-enabled httpd
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
curl -s http://localhost/
ls -Z /var/www/html/index.html
ls -ld /srv/compartido
sudo -u dev01 touch /srv/compartido/ticket-t2; ls -l /srv/compartido
df -h /backups
sudo /usr/local/bin/backup.sh /home/student/empresa /backups
cat /etc/cron.d/backup-empresa
sudo passwd -S dev02
getent passwd dev02
```
Más las dos capturas del navegador. Si se quiere confirmar la persistencia: `sudo reboot` y repetir el bloque.

## Errores que se ven

- Empezar por el firewall sin mirar `systemctl status`: "primero el servicio, después la puerta".
- Abrir `http` solo en `public` y declararlo resuelto: `localhost:8080` carga, `192.168.56.10` no. El ticket sigue abierto.
- `systemctl start` sin `enable`: funciona hoy y vuelve a fallar en el próximo reinicio. Pedir `is-enabled`.
- T5 con `chcon -t httpd_sys_content_t`: funciona hoy y se pierde con el próximo relabel. Preguntar "¿y después de un `touch /.autorelabel` como el de la mañana?"; pedir `restorecon`.
- T3: borrar los `.tar.gz` legítimos porque `ls /backups` no muestra el oculto. Recordar `ls -la`.
- T3: `rm .cache_old.img` y `df` sigue en `100%`: alguien lo tenía abierto con `less` o `tail`. Es el Lab 1.1: `sudo lsof +L1`.
- T4: `passwd -u` y declarar resuelto sin probar `su - dev02`: la shell sigue en `nologin`. Y si probó cinco veces con la contraseña equivocada, faillock: ahora hay tres causas.
- Restaurar el snapshot y decir "resuelto": el snapshot reinicia el intento, no resuelve nada.
