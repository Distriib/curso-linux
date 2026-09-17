# Lab 2 — Servicio propio y hora del sistema

Vamos a escribir un servicio desde cero, `monitor.service`, que anota la carga cada 30 segundos; habilitarlo, matarlo y verlo levantarse solo; y comprobar la hora.

---

## Parte 1 — El programa que va a correr el servicio

**¿Qué se prueba antes de convertir algo en servicio?**

```bash
sudo vim /usr/local/bin/monitor.sh
```

`i`, pegar esto, `Esc`, `:wq`:

```
#!/bin/bash
while true
do
    uptime
    uptime | logger -t monitor -p local0.info
    sleep 30
done
```

```bash
sudo chmod 755 /usr/local/bin/monitor.sh
ls -l /usr/local/bin/monitor.sh
timeout 3 /usr/local/bin/monitor.sh
```

Foto.

**Comprobar:**
```
-rwxr-xr-x. 1 root root 96 ... /usr/local/bin/monitor.sh
 10:25:10 up 1:23,  1 user,  load average: 0.12, 0.20, 0.15
```
Una línea, y a los 3 segundos vuelve el prompt. El programa escribe dos veces: a la pantalla (systemd lo va a capturar) y al log del sistema con `logger`.

---

## Parte 2 — El unit file

```bash
sudo vim /etc/systemd/system/monitor.service
```

`i`, pegar esto, `Esc`, `:wq`:

```
[Unit]
Description=Monitor de carga (curso Dia 04)
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemd-analyze verify /etc/systemd/system/monitor.service
sudo systemctl daemon-reload
```

Foto.

**Comprobar:** `verify` no imprime nada. `daemon-reload` tampoco.

---

## Parte 3 — Habilitar y arrancar

```bash
sudo systemctl enable --now monitor.service
systemctl status monitor
```

Foto.

**Comprobar:**
```
Created symlink /etc/systemd/system/multi-user.target.wants/monitor.service → /etc/systemd/system/monitor.service.
● monitor.service - Monitor de carga (curso Dia 04)
     Loaded: loaded (/etc/systemd/system/monitor.service; enabled; preset: disabled)
     Active: active (running) since ...
   Main PID: 5120 (monitor.sh)
      Tasks: 2 (limit: 23000)
     CGroup: /system.slice/monitor.service
             ├─5120 /bin/bash /usr/local/bin/monitor.sh
             └─5133 sleep 30

Sep 04 10:27:02 rhel01 systemd[1]: Started Monitor de carga (curso Dia 04).
Sep 04 10:27:02 rhel01 monitor.sh[5120]:  10:27:02 up 1:25,  1 user,  load average: 0.10, 0.18, 0.14
Sep 04 10:27:02 rhel01 monitor[5121]:  10:27:02 up 1:25,  1 user,  load average: 0.10, 0.18, 0.14
```
`preset: disabled` es normal para unidades propias. En `CGroup` están el programa y su `sleep`: los dos son "el servicio".

---

## Parte 4 — Seguir el log de la unidad

```bash
journalctl -u monitor -n 3 --no-pager
journalctl -u monitor -f
```

Esperar a que aparezca una línea nueva. Ctrl+C. Foto.

**Comprobar:** las últimas 3 líneas, y después una nueva cada 30 segundos. `-u` filtra por unidad, `-f` sigue en vivo.

---

## Parte 5 — Matarlo y verlo levantarse solo

**¿Qué hace `Restart=on-failure` cuando el proceso muere mal?**

```bash
pgrep -a monitor.sh
sudo pkill -9 monitor.sh
sleep 7
systemctl status monitor --no-pager | head -6
journalctl -u monitor -n 8 --no-pager | grep systemd
```

Foto.

**Comprobar:**
```
5120 /bin/bash /usr/local/bin/monitor.sh
     Active: active (running) since ...; 2s ago
   Main PID: 5210 (monitor.sh)
Sep 04 10:29:05 rhel01 systemd[1]: monitor.service: Main process exited, code=killed, status=9/KILL
Sep 04 10:29:05 rhel01 systemd[1]: monitor.service: Failed with result 'signal'.
Sep 04 10:29:10 rhel01 systemd[1]: monitor.service: Scheduled restart job, restart counter is at 1.
Sep 04 10:29:10 rhel01 systemd[1]: Started Monitor de carga (curso Dia 04).
```
Murió con señal 9, systemd esperó 5 segundos y lo relanzó con PID nuevo. Un `systemctl stop` limpio **no** dispara el reinicio. Esto es lo que `nohup` jamás va a dar.

---

## Parte 6 — La hora

**¿Por qué la hora es lo primero que se revisa antes de leer un log?**

```bash
timedatectl
chronyc sources
```

Foto.

**Comprobar:**
```
               Local time: Thu 2026-09-04 10:31:12 EST
           Universal time: Thu 2026-09-04 15:31:12 UTC
                Time zone: America/Panama (EST, -0500)
System clock synchronized: yes
              NTP service: active
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
 * 129.6.15.28                   1   6   377    35  -1201us[-1357us] +/-   27ms
 + 200.29.147.31                 2   6   377    36   +850us[ +850us] +/-   61ms
```
`synchronized: yes` y `NTP service: active` es lo que hay que ver en todo servidor. Los servidores de hora son distintos en cada VM; el elegido lleva un `*` en la segunda columna.
