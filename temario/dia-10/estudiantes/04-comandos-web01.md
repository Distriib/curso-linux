# 4 — `web01`: qué tiene y cómo se verifica cada pieza

El servidor del reto es el resultado del curso. Se construye igual en todas las VMs para que todos tengamos **el mismo servidor**; se verifica pieza por pieza; después, snapshot. Recién entonces se rompe.

| Pieza | Día | Cómo se verifica |
|---|---|---|
| Nombre `web01.lab.local` | 5 | `hostname` |
| `dev01` y `dev02` en el grupo `sistemas`; `/srv/compartido` con herencia de grupo (`2770 root:sistemas`) | 3 | `id dev01` · `ls -ld /srv/compartido` |
| Apache en el 80 sirviendo `/var/www/html/index.html`, activo **y habilitado** | 4, 9 | `systemctl is-active httpd` · `systemctl is-enabled httpd` · `curl -s http://localhost/` |
| `http` abierto en las **dos** zonas del firewall | 8 | `sudo firewall-cmd --list-services` · `sudo firewall-cmd --zone=internal --list-services` |
| `index.html` con contexto `httpd_sys_content_t` | 8 | `ls -Z /var/www/html/index.html` |
| `/backups` en su propio volumen; `backup.sh` en `/usr/local/bin`; programado cada noche | 6, 7 | `df -h /backups` · `sudo /usr/local/bin/backup.sh /home/student/empresa /backups` · `cat /etc/cron.d/backup-empresa` |

```bash
hostname
sudo firewall-cmd --get-active-zones
curl -s http://localhost/
```

## Las dos zonas

```
computadora del participante  →  http://localhost:8080/   →  NAT (10.0.2.x)   →  zona public
computadora del participante  →  http://192.168.56.10/    →  host-only        →  zona internal
```

Un servicio publicado solo en `public` "funciona" desde `localhost:8080` y **falla desde la red de la oficina**. Todo lo que se publica va en las dos.

## Verificar desde donde lo ve el usuario

| Desde | Qué |
|---|---|
| el navegador de su computadora | `http://192.168.56.10/` **y** `http://localhost:8080/` |
| otra cuenta | `sudo -u dev01 touch /srv/compartido/prueba` |
| el propio script | `sudo /usr/local/bin/backup.sh /home/student/empresa /backups` termina en `OK: creado` |
| después de un reinicio | todo lo anterior, otra vez |
