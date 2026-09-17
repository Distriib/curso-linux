# Reto — Servidor `web01` averiado

**Solos, sin ayuda.** Solo `man`, los apuntes de los diez días y el método del Bloque 1. Lo que no se termina en clase queda de tarea.

## Preparar el servidor averiado

1. Confirmar que existe el snapshot `dia10-pre-romper`.
2. Guardar el archivo `romper.sh` que el instructor pega en el chat: `vim romper.sh`, `i`, pegar, `Esc`, `:wq`.
3. Ejecutarlo: `sudo bash romper.sh`.

Desde ese momento, llegan los tickets.

## Los tickets

> **T1** — "La página web de la Procuraduría no carga desde las computadoras de la oficina. Ayer sí funcionaba. Es urgente, la publica Comunicación Social."

> **T2** — "Soy dev01. Desde esta mañana no puedo guardar nada en la carpeta compartida `/srv/compartido`. Mi compañero dev02 tampoco. Dice 'Permiso denegado'."

> **T3** — "El respaldo de anoche falló. En el correo de error dice algo de 'No space left'. Necesitamos que el respaldo vuelva a funcionar hoy y que se ejecute solo cada noche."

> **T4** — "dev02 no puede iniciar sesión en el servidor. Probó con su contraseña varias veces y nada. Está seguro de que es la correcta."

> **T5** — "Después de que 'arreglaron' la web, ahora muestra 'Forbidden' (403). El archivo `index.html` está ahí, lo acabo de ver, y tiene permisos de lectura."

Orden sugerido: T1 → T5 (aparece al resolver T1) → T2 → T3 → T4.

## Entregable por ticket (4 líneas)

qué estaba mal · cómo lo encontré (comando) · cómo lo corregí (comando) · cómo lo verifiqué (desde donde lo ve el usuario)

**Prohibido:** `setenforce 0`, `systemctl stop firewalld`, `chmod 777`, `chcon` como solución final, restaurar el snapshot como "solución" (sí para empezar el intento de nuevo).

La corrección tiene que ser **persistente**: aguanta un `sudo reboot`.

## Verificación

Pegar en el chat la salida completa de:

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

Está bien si: `active` y `enabled`; `http` en **las dos** listas; `<h1>Servidor web01 - Procuraduria General de la Nacion</h1>`; `httpd_sys_content_t`; `drwxrws---. 2 root sistemas`; `ticket-t2` con grupo `sistemas`; `/backups` lejos del 100 %; `OK: creado`; las tres líneas del cron; `dev02 PS`; `/bin/bash` al final. Más las dos capturas del navegador: `http://192.168.56.10/` y `http://localhost:8080/`.
