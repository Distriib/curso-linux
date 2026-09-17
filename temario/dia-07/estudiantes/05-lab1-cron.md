# Lab 1 — `cron`

Vamos a caer en la trampa del `PATH` y salir de ella, dejar el `backup.sh` programado todas las noches, y ver quién puede usar `crontab`.

| Tarea | Cuándo |
|---|---|
| `backup.sh /home/student/empresa` | todos los días a las 23:30 |
| `hola.sh` | cada 15 minutos, de 8 a 17 h, lunes a viernes |

---

## Parte 1 — Cómo está `cron` en este servidor

**¿Está corriendo? ¿Qué archivos tiene? ¿Tengo algo programado?**

```bash
systemctl is-active crond
ls -d /etc/cron*
crontab -l
```

**Comprobar:**
```
active
/etc/cron.d  /etc/cron.daily  /etc/cron.deny  /etc/cron.hourly  /etc/cron.monthly  /etc/crontab  /etc/cron.weekly
no crontab for student
```

---

## Parte 2 — Dos tareas cada minuto

**¿Cómo instalo un crontab desde un archivo?**

```bash
cat > ~/crontab-prueba <<'EOF'
* * * * * date >> /home/student/cron-prueba.txt
* * * * * hola.sh >> /home/student/cron-hola.txt 2>&1
EOF
crontab ~/crontab-prueba
crontab -l
```

**Comprobar:**
```
* * * * * date >> /home/student/cron-prueba.txt
* * * * * hola.sh >> /home/student/cron-hola.txt 2>&1
```

---

## Parte 3 — La trampa del `PATH`

**Esperar a que cambie el minuto. ¿Las dos tareas funcionaron?**

```bash
sudo tail -3 /var/log/cron
cat ~/cron-prueba.txt
cat ~/cron-hola.txt
```

**Comprobar:**
```
Sep 16 10:46:01 rhel01 CROND[6210]: (student) CMD (date >> /home/student/cron-prueba.txt)
Sep 16 10:46:01 rhel01 CROND[6211]: (student) CMD (hola.sh >> /home/student/cron-hola.txt 2>&1)
...
Tue Sep 16 10:46:01 AM EST 2026
/bin/sh: line 1: hola.sh: command not found
```
`cron` **sí** ejecutó `hola.sh` (está en el log), pero su shell no lo encontró: `cron` no conoce `~/bin`. A mano funciona; en `cron`, no. El script es el mismo; el entorno, no.

---

## Parte 4 — El crontab definitivo

**¿Cómo se escribe un crontab que no cae en la trampa?**

```bash
cat > ~/mi-crontab <<'EOF'
SHELL=/bin/bash
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""
30 23 * * *        /home/student/bin/backup.sh /home/student/empresa >> /home/student/backup.log 2>&1
*/15 8-17 * * 1-5  hola.sh >> /home/student/hola.log 2>&1
EOF
crontab ~/mi-crontab
crontab -l
rm -f ~/cron-prueba.txt ~/cron-hola.txt ~/crontab-prueba
```

Ahora ustedes: agreguen a `~/mi-crontab` una línea que ejecute `/home/student/bin/revisar.sh /home/student/empresa` los lunes a las 08:00, con la salida en `/home/student/revisar.log`; instalen el archivo otra vez y muestren `crontab -l`. Foto.

**Comprobar:**
```
SHELL=/bin/bash
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
MAILTO=""
30 23 * * *        /home/student/bin/backup.sh /home/student/empresa >> /home/student/backup.log 2>&1
*/15 8-17 * * 1-5  hola.sh >> /home/student/hola.log 2>&1
```
`crontab ARCHIVO` reemplaza **todo** el crontab anterior: las tareas de cada minuto desaparecieron.

---

## Parte 5 — El `cron` del sistema

**¿Dónde programa sus tareas el propio RHEL?**

```bash
cat /etc/crontab
cat /etc/cron.d/0hourly
ls /etc/cron.hourly /etc/cron.daily
```

**Comprobar:**
```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
...
# *  *  *  *  * user-name  command to be executed
...
01 * * * * root run-parts /etc/cron.hourly
/etc/cron.hourly:
0anacron
/etc/cron.daily:
```
`/etc/crontab` no tiene tareas: solo variables y el formato. En `0hourly`, el sexto campo es `root`: el usuario que ejecuta. `/etc/cron.daily` puede estar vacío.

---

## Parte 6 — Quién puede usar `crontab`

**¿Cómo le prohíbo `crontab` a un usuario?**

Primero le quitamos a `jperez` el cambio de contraseña pendiente (`crontab` no atiende a cuentas en ese estado):

```bash
sudo chage -d "$(date +%F)" jperez
cat /etc/cron.deny
echo jperez | sudo tee /etc/cron.deny
sudo -u jperez crontab -l
sudo truncate -s 0 /etc/cron.deny
sudo -u jperez crontab -l
```

**Comprobar:**
```
jperez
You (jperez) are not allowed to use this program (crontab)
See crontab(1) for more information
no crontab for jperez
```
El primer `cat` no muestra nada: el archivo existe vacío. Con `jperez` adentro, rechazo. Vacío otra vez, normal.
