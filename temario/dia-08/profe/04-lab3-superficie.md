# Lab — Superficie: qué escucha, qué se cierra, parches solos (comandos)

Ocho minutos. Vos primero, ellos después, foto.

## Parte 1 — Qué corre y qué escucha
```bash
systemctl list-units --type=service --state=running --no-pager
sudo ss -tulpn
```
Qué decir: "cada `LISTEN` en `0.0.0.0` o `*` es una puerta a la red. `chronyd` escucha en `127.0.0.1`: solo desde adentro, bien. Lo que no se usa: `sudo systemctl disable --now servicio`." Hoy no apagamos ninguno: todos los que corren se usan.
Si alguien activó Cockpit en la tarea del Día 7, va a ver además el 9090 escuchando: es esperado.

## Parte 2 — Cerrar lo que nadie pidió
```bash
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --permanent --zone=internal --remove-service=cockpit
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```
Qué decir: "cockpit venía abierto de fábrica aunque no esté instalado. Nadie lo pidió: se cierra. Si un día hace falta, se abre a propósito, en la zona que corresponda." Quien lo use por `localhost:9090` deja de poder: es el precio, y el Día 10 muestra cómo volverlo a abrir.

## Parte 3 — Parches de seguridad solos
```bash
sudo vim /etc/dnf/automatic.conf
```
`/apply_updates` `Enter` → `apply_updates = yes`; `/upgrade_type` `Enter` → `upgrade_type = security`; `Esc` `:wq`.
```bash
grep apply_updates /etc/dnf/automatic.conf
grep upgrade_type /etc/dnf/automatic.conf
sudo systemctl enable --now dnf-automatic.timer
systemctl list-timers dnf-automatic.timer --no-pager
```
Qué decir: "`security` = solo erratas de seguridad, menos riesgo de que un cambio funcional rompa algo. No reinicia el servidor: eso se planifica." El paquete trae varios timers (`dnf-automatic-install.timer`, `-download`, `-notifyonly`); se habilita **uno solo**, el que obedece al archivo: `dnf-automatic.timer`.

## Parte 4 — Lo que ya queda registrado
```bash
sudo grep COMMAND /var/log/secure | tail -3
```
Qué decir: "cada `sudo` del día quedó acá: quién, desde dónde, qué. Y auditd guardó cada intento fallido de `su` del lab anterior. Sin configurar nada. Esto es lo primero que se mira después de un incidente."
