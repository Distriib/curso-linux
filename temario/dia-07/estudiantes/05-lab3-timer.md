# Lab 5.3 — Temporizador de systemd

Vamos a dejar `monitor-disco.sh` corriendo cada 10 minutos, ejecutado por systemd como root. Donde dice **Ahora ustedes**, el comando no está escrito: hay que resolverlo y mandar foto. La solución está al final de la hoja.

| Archivo | Qué |
|---|---|
| `/etc/systemd/system/monitor-disco.service` | qué se ejecuta: el script con umbral 80 |
| `/etc/systemd/system/monitor-disco.timer` | cuándo: cada 10 minutos, recuperando lo perdido |

---

## Parte 1 — Los temporizadores que RHEL ya trae

**¿Qué tareas programadas tiene el sistema aunque nadie las haya puesto?**

```bash
systemctl list-timers --no-pager | head -5
```

**Comprobar:**
```
NEXT                        LEFT       LAST                        PASSED    UNIT                         ACTIVATES
Tue 2026-09-16 10:12:33 EST 7min left  Tue 2026-09-16 09:12:33 EST 52min ago dnf-makecache.timer          dnf-makecache.service
...                                                                          systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
...                                                                          logrotate.timer              logrotate.service
```

---

## Parte 2 — El service: qué se ejecuta

**¿Cómo se escribe un archivo en `/etc` con `sudo`?**

```bash
sudo tee /etc/systemd/system/monitor-disco.service <<'EOF'
[Unit]
Description=Monitor de uso de disco (PGN)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-disco.sh 80
EOF
```

**Comprobar:** `tee` repite en pantalla las seis líneas que guardó.

---

## Parte 3 — El timer: cuándo

**¿Cómo se activa un temporizador?**

```bash
sudo tee /etc/systemd/system/monitor-disco.timer <<'EOF'
[Unit]
Description=Ejecuta monitor-disco cada 10 minutos

[Timer]
OnCalendar=*:0/10
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now monitor-disco.timer
systemctl list-timers monitor-disco.timer --no-pager
```

**Comprobar:**
```
Created symlink /etc/systemd/system/timers.target.wants/monitor-disco.timer → /etc/systemd/system/monitor-disco.timer.
NEXT                        LEFT      LAST PASSED UNIT                ACTIVATES
Tue 2026-09-16 10:10:00 EST 4min left n/a  n/a    monitor-disco.timer monitor-disco.service
```
`LAST` dice `n/a`: todavía no corrió ninguna vez.

---

## Parte 4 — Ejecutarlo ahora, sin esperar

**¿Cómo pruebo el service sin esperar los 10 minutos?**

```bash
sudo systemctl start monitor-disco.service
sudo journalctl -u monitor-disco.service -n 4 --no-pager
systemctl status monitor-disco.timer --no-pager | head -5
```

**Comprobar:**
```
Sep 16 10:06:12 rhel01 systemd[1]: Starting Monitor de uso de disco (PGN)...
Sep 16 10:06:12 rhel01 monitor-disco.sh[6810]: Revisados 5 sistemas de archivos, 0 alerta(s)
Sep 16 10:06:12 rhel01 systemd[1]: monitor-disco.service: Deactivated successfully.
Sep 16 10:06:12 rhel01 systemd[1]: Finished Monitor de uso de disco (PGN).
● monitor-disco.timer - Ejecuta monitor-disco cada 10 minutos
     Loaded: loaded (/etc/systemd/system/monitor-disco.timer; enabled; preset: disabled)
     Active: active (waiting) since Tue 2026-09-16 10:05:40 EST; 1min ago
    Trigger: Tue 2026-09-16 10:10:00 EST; 3min left
   Triggers: ● monitor-disco.service
```
Lo que el script imprimió quedó en el journal sin redirigir nada.

---

## Parte 5 — Probar una expresión de calendario antes de usarla

**¿Cómo sé que `Mon..Fri 08:00` significa lo que creo?**

```bash
systemd-analyze calendar "Mon..Fri 08:00"
systemd-analyze calendar --iterations=3 "*:0/10"
```

Ahora ustedes: la expresión para "sábados a las 23:00" y la de "el día 1 de cada mes a las 03:00". Foto.

**Comprobar:**
```
  Original form: Mon..Fri 08:00
Normalized form: Mon..Fri *-*-* 08:00:00
    Next elapse: Wed 2026-09-17 08:00:00 EST
       (in UTC): Wed 2026-09-17 13:00:00 UTC
       From now: 21h left
...
Normalized form: *-*-* *:00/10:00
    Next elapse: Tue 2026-09-16 10:10:00 EST
       Iter. #2: Tue 2026-09-16 10:20:00 EST
       Iter. #3: Tue 2026-09-16 10:30:00 EST
```

---

# Solución — todos los comandos

```bash
# Parte 1 — los temporizadores que ya trae RHEL
systemctl list-timers --no-pager | head -5

# Parte 2 — el service: QUÉ se ejecuta
sudo tee /etc/systemd/system/monitor-disco.service <<'EOF'
[Unit]
Description=Monitor de uso de disco (PGN)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-disco.sh 80
EOF

# Parte 3 — el timer: CUÁNDO
sudo tee /etc/systemd/system/monitor-disco.timer <<'EOF'
[Unit]
Description=Ejecuta monitor-disco cada 10 minutos

[Timer]
OnCalendar=*:0/10
Persistent=true

[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now monitor-disco.timer
systemctl list-timers monitor-disco.timer --no-pager

# Parte 4 — ejecutarlo ahora, sin esperar
sudo systemctl start monitor-disco.service
sudo journalctl -u monitor-disco.service -n 4 --no-pager
systemctl status monitor-disco.timer --no-pager | head -5

# Parte 5 — probar expresiones de calendario
systemd-analyze calendar "Mon..Fri 08:00"
systemd-analyze calendar --iterations=3 "*:0/10"
```

**Parte 5 (Ahora ustedes)** — las dos expresiones que pedía:
```bash
systemd-analyze calendar "Sat *-*-* 23:00"
systemd-analyze calendar "*-*-01 03:00"
```
La primera normaliza a `Sat *-*-* 23:00:00`; la segunda, a `*-*-01 03:00:00`. `systemd-analyze calendar` sirve para comprobar una expresión **antes** de ponerla en un timer: si está mal escrita, da `Failed to parse`.

Lo que se habilita es **el timer**, no el service: `systemctl enable --now monitor-disco.timer`. El service se ejecuta solo cuando el timer lo dispara (o a mano con `systemctl start`).
