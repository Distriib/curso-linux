# Lab — Modos y contextos (comandos)

Vos primero, ellos después, foto. Apache tiene que estar caído desde el Lab 1.2 (con `/etc/httpd/conf.d/puerto.conf`); si alguien lo arregló por su cuenta, que vuelva a crear el archivo con el `echo | sudo tee` de ese lab.

## Parte 1 — En qué modo estamos
```bash
getenforce
sestatus
```

## Parte 2 — La prueba del permissive
```bash
sudo setenforce 0
getenforce
sudo systemctl restart httpd
curl -I http://localhost:82
sudo ausearch -m AVC -ts recent | tail -1
```
Qué decir: "arrancó. Confirmado: era SELinux. Y miren la última palabra: `permissive=1`: en permissive no bloquea, pero registra igual. Esto se hace por unos segundos; nunca se deja así."
`ausearch -m AVC -ts recent`: busca en el log de auditoría (`/var/log/audit/audit.log`, Día 4) los registros de tipo AVC (denegaciones de SELinux) de los últimos 10 minutos. Se explica completo en el Bloque 3; acá solo "miren `denied`, `httpd`, `82`".

## Parte 3 — Volver a enforcing y dejar Apache sano
```bash
sudo setenforce 1
sudo systemctl restart httpd
sudo rm /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
systemctl is-active httpd
grep -v "#" /etc/selinux/config
```
El primer `restart` falla a propósito (enforcing, 82). Después de borrar el archivo, arranca en el 80. `grep -v "#"` (Día 2): las líneas que **no** tienen `#`, o sea las dos de configuración real.
Qué decir: "no tocamos `/etc/selinux/config`: el arranque sigue en `enforcing`. El 82 se arregla bien después del descanso."

## Parte 4 — Leer etiquetas
```bash
id -Z
ps -eZ | grep httpd
ls -Z /var/www/html/
ls -Zd /root /home/student /etc/shadow /var/www/html /tmp
```
Si `index.html` sale con `system_u` en vez de `unconfined_u` en alguna VM, no importa: el tipo es el mismo. Decirlo antes de que pregunten.
