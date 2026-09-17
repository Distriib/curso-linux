# Reto — Solución

Datos de prueba (los ejecutan todos antes):
```bash
mkdir -p ~/empresa/logs/app
cd ~/empresa/logs/app
for i in {1..5}; do echo "log reciente $i" > reciente-$i.log; done
for i in {1..5}; do echo "log viejo $i" > viejo-$i.log; touch -d "-10 days" viejo-$i.log; done
for i in {1..3}; do echo "gz antiguo $i" | gzip > antiguo-$i.log.gz; touch -d "-45 days" antiguo-$i.log.gz; done
ls -l --time-style=+%F
cd
```

## 1, 2 y 3 — El script
```bash
cat > /tmp/limpiar-logs.sh <<'EOF'
#!/bin/bash
# limpiar-logs.sh - comprime .log viejos y borra .gz de más de 30 días
# Uso:    limpiar-logs.sh CARPETA DIAS
# Salida: 0 ok · 1 argumentos inválidos
set -e

DIR="$1"
DIAS="$2"
RETENCION_GZ=30

if [[ $# -ne 2 ]]; then
    echo "Uso: $0 CARPETA DIAS" >&2
    exit 1
fi
if [[ ! -d "$DIR" ]]; then
    echo "ERROR: la carpeta $DIR no existe" >&2
    exit 1
fi

comprimidos=0
for archivo in $(find "$DIR" -maxdepth 1 -name "*.log" -mtime +"$DIAS"); do
    gzip "$archivo"
    comprimidos=$((comprimidos + 1))
done

borrados=$(find "$DIR" -maxdepth 1 -name "*.gz" -mtime +$RETENCION_GZ -print -delete | wc -l)

logger -t limpiar-logs "$DIR: $comprimidos .log comprimidos, $borrados .gz borrados"
echo "$(date '+%F %T') $DIR: $comprimidos comprimidos, $borrados borrados"
EOF
sudo cp /tmp/limpiar-logs.sh /usr/local/bin/limpiar-logs.sh
sudo chmod 755 /usr/local/bin/limpiar-logs.sh
bash -n /usr/local/bin/limpiar-logs.sh && echo "sintaxis ok"
```
`-maxdepth 1`: solo esa carpeta, sin subcarpetas. `gzip` conserva la fecha del original: los `viejo-*.log.gz` siguen teniendo 10 días y no se borran hasta cumplir 30.

## 4 — cron
```bash
echo "0 2 * * 1-5  /usr/local/bin/limpiar-logs.sh /home/student/empresa/logs/app 7 >> /home/student/limpiar-logs.log 2>&1" >> ~/mi-crontab
crontab ~/mi-crontab
crontab -l | tail -1
```

## 4 — Temporizador de systemd
```bash
sudo tee /etc/systemd/system/limpiar-logs.service <<'EOF'
[Unit]
Description=Limpieza de logs de expedientes (PGN)

[Service]
Type=oneshot
User=student
ExecStart=/usr/local/bin/limpiar-logs.sh /home/student/empresa/logs/app 7
EOF
sudo tee /etc/systemd/system/limpiar-logs.timer <<'EOF'
[Unit]
Description=Ejecuta limpiar-logs de lunes a viernes a las 02:00

[Timer]
OnCalendar=Mon..Fri 02:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now limpiar-logs.timer
```
`User=student`: el service corre como `student`, así los `.gz` siguen siendo suyos. Es algo que el `cron` de usuario hace solo y al timer hay que decírselo.

## Verificación
```bash
limpiar-logs.sh; echo "código: $?"
limpiar-logs.sh /no/existe 7; echo "código: $?"
limpiar-logs.sh /home/student/empresa/logs/app 7
ls ~/empresa/logs/app
crontab -l | tail -1
systemctl list-timers limpiar-logs.timer --no-pager | head -2
sudo journalctl -t limpiar-logs -n 1 --no-pager
```
Salida esperada:
```
Uso: /usr/local/bin/limpiar-logs.sh CARPETA DIAS
código: 1
ERROR: la carpeta /no/existe no existe
código: 1
2026-09-16 11:02:35 /home/student/empresa/logs/app: 5 comprimidos, 3 borrados
reciente-1.log  reciente-3.log  reciente-5.log  viejo-2.log.gz  viejo-4.log.gz
reciente-2.log  reciente-4.log  viejo-1.log.gz  viejo-3.log.gz  viejo-5.log.gz
0 2 * * 1-5  /usr/local/bin/limpiar-logs.sh /home/student/empresa/logs/app 7 >> /home/student/limpiar-logs.log 2>&1
NEXT                        LEFT     LAST PASSED UNIT               ACTIVATES
Wed 2026-09-17 02:00:00 EST 14h left n/a  n/a    limpiar-logs.timer limpiar-logs.service
Sep 16 11:02:35 rhel01 limpiar-logs[7420]: /home/student/empresa/logs/app: 5 .log comprimidos, 3 .gz borrados
```

## Errores que se ven

- El script quedó en `~/bin` y no en `/usr/local/bin`: a mano funciona, el timer falla con `203/EXEC`. Copiarlo con `cp` a `/usr/local/bin`.
- `crontab ~/mi-crontab` sin haber agregado la línea al archivo: el crontab queda igual. Mirar `crontab -l`.
- Timer sin `User=student`: corre como root y los `.gz` quedan de root. Funciona; señalarlo.
- Sin `-maxdepth 1`: funciona igual (no hay subcarpetas). Aceptar.
- La segunda ejecución da `0 comprimidos, 0 borrados`: correcto, ya no queda nada viejo.
- Cerrar: en producción se deja **una** de las dos programaciones, no las dos (correría dos veces). Preguntar cuál elegirían.
