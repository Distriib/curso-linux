# Reto — Ticket PGN-1187

**Solos, sin ayuda.** Si no se termina en clase, queda de tarea.

> **Limpieza de logs de la aplicación de expedientes**
> Solicitante: Dirección de Informática. Servidor: `rhel01`.
> "La carpeta `/home/student/empresa/logs/app` se llena de archivos `.log`. Necesitamos un script que los comprima cuando son viejos, borre los comprimidos muy viejos, y corra solo de lunes a viernes a las 02:00."

## Datos de prueba

Todos ejecutan lo mismo antes de empezar:

```bash
mkdir -p ~/empresa/logs/app
cd ~/empresa/logs/app
for i in {1..5}; do echo "log reciente $i" > reciente-$i.log; done
for i in {1..5}; do echo "log viejo $i" > viejo-$i.log; touch -d "-10 days" viejo-$i.log; done
for i in {1..3}; do echo "gz antiguo $i" | gzip > antiguo-$i.log.gz; touch -d "-45 days" antiguo-$i.log.gz; done
ls -l --time-style=+%F
cd
```

Tienen que quedar 13 archivos: 5 de hoy, 5 de hace 10 días y 3 de hace 45 días.

## Lo que piden

1. El script `/usr/local/bin/limpiar-logs.sh` recibe **dos argumentos**: la carpeta y una cantidad de días. Si falta alguno, o la carpeta no existe, muestra un mensaje de error y termina con código `1`.
2. Comprime con `gzip` los `.log` de la carpeta que tengan más de esa cantidad de días, y borra los `.gz` que tengan más de **30 días**.
3. Registra con `logger` (etiqueta `limpiar-logs`) cuántos archivos comprimió y cuántos borró.
4. Queda programado a las **02:00 de lunes a viernes**, con 7 días como argumento, de dos formas: una línea en el crontab de `student` **y** un temporizador de systemd `limpiar-logs.timer` (el service corre como `student`).

## Verificación

Pegar en el chat la salida completa de:

```bash
limpiar-logs.sh; echo "código: $?"
limpiar-logs.sh /no/existe 7; echo "código: $?"
limpiar-logs.sh /home/student/empresa/logs/app 7
ls ~/empresa/logs/app
crontab -l | tail -1
systemctl list-timers limpiar-logs.timer --no-pager | head -2
sudo journalctl -t limpiar-logs -n 1 --no-pager
```

Está bien si: los dos primeros dan `código: 1` con un mensaje; después de la ejecución real quedan `reciente-1.log` a `reciente-5.log` y `viejo-1.log.gz` a `viejo-5.log.gz`, y ningún `antiguo-`; la línea del crontab empieza con `0 2 * * 1-5`; `list-timers` muestra `limpiar-logs.timer` con `NEXT` a las `02:00` de un día hábil; el journal dice `5 .log comprimidos, 3 .gz borrados`.
