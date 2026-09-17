# Lab 3 — Temporizador de systemd (comandos)

Pegar los dos archivos de unidad en el chat.

## Parte 1 — Los temporizadores que RHEL ya trae
```bash
systemctl list-timers --no-pager | head -5
```

## Parte 2 — El service: qué se ejecuta
```bash
sudo tee /etc/systemd/system/monitor-disco.service <<'EOF'
[Unit]
Description=Monitor de uso de disco (PGN)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor-disco.sh 80
EOF
```
Decir: no tiene `[Install]` y no se habilita: lo dispara el timer.

## Parte 3 — El timer: cuándo
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
Si el timer no aparece: falta `daemon-reload`, o el archivo tiene un error de nombre. `systemctl status monitor-disco.timer` lo dice.

## Parte 4 — Ejecutarlo ahora, sin esperar
```bash
sudo systemctl start monitor-disco.service
sudo journalctl -u monitor-disco.service -n 4 --no-pager
systemctl status monitor-disco.timer --no-pager | head -5
```
Si el journal dice `status=203/EXEC`: el script no tiene `x` o se movió con `mv`. `sudo chmod 755 /usr/local/bin/monitor-disco.sh` y `sudo restorecon -v /usr/local/bin/monitor-disco.sh`.

## Parte 5 — Probar una expresión de calendario antes de usarla
```bash
systemd-analyze calendar "Mon..Fri 08:00"
systemd-analyze calendar --iterations=3 "*:0/10"
```
Ellos:
```bash
systemd-analyze calendar "Sat *-*-* 23:00"
systemd-analyze calendar "*-*-01 03:00"
```
