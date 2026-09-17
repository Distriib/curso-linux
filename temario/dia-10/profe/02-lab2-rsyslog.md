# Lab — rsyslog: regla propia y colector por red (comandos)

Yo hago el primero, ellos los demás, foto. Los dos archivos se escriben con `vim`: `i`, pegar, `Esc`, `:wq`.

## Parte 1
```bash
grep module /etc/rsyslog.conf
grep include /etc/rsyslog.conf
grep /var/log /etc/rsyslog.conf
ls /etc/rsyslog.d/
sudo rsyslogd -N1
```

## Parte 2
```bash
sudo vim /etc/rsyslog.d/pgn-app.conf
```
```
# Aplicacion institucional (facilidad local5)
local5.err      /var/log/messages
local5.*        /var/log/pgn-app.log
local5.*        stop
```
```bash
sudo rsyslogd -N1
sudo systemctl restart rsyslog
logger -p local5.info -t portal-pgn "Usuario ana consulto expediente 2026-001"
logger -p local5.err -t portal-pgn "Fallo de conexion a la base de datos"
sudo tail -2 /var/log/pgn-app.log
sudo grep portal-pgn /var/log/messages
```
Ellos: `logger -p local5.warning -t portal-pgn "Aviso de NOMBRE"` y `sudo tail -1 /var/log/pgn-app.log`.
Si rsyslog no arranca después de editar: `sudo rsyslogd -N1` dice archivo y línea del error (casi siempre una comilla o un paréntesis).

## Parte 3
```bash
sudo vim /etc/rsyslog.d/remoto.conf
```
```
# Colector: recibe por TCP 514 y lo guarda aparte
module(load="imtcp")
ruleset(name="desde_remotos") {
    action(type="omfile" file="/var/log/remoto.log")
}
input(type="imtcp" port="514" ruleset="desde_remotos")
```
```bash
sudo rsyslogd -N1
sudo semanage port -l | grep syslogd_port_t
sudo semanage port -m -t syslogd_port_t -p tcp 514
sudo semanage port -l | grep syslogd_port_t
sudo systemctl restart rsyslog
sudo ss -tlnp | grep 514
```
Si `ss` no muestra el 514: `sudo ausearch -m AVC -ts recent | grep rsyslogd` (buscar `name_bind`) y `sudo journalctl -u rsyslog -n 5`. Casi siempre es que faltó el `semanage port -m` o el `restart`.

## Parte 4
```bash
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=514/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
```
Decir: "para probar desde la misma VM el firewall no interviene; se abre porque un colector real recibe de afuera, y en las dos zonas por la regla del Día 8."

## Parte 5
```bash
logger -n 127.0.0.1 -P 514 -T -t prueba-directa "Mensaje enviado por TCP al colector"
sudo tail -2 /var/log/remoto.log
```
Ellos: `logger -n 192.168.56.10 -P 514 -T -t prueba-NOMBRE "hola desde NOMBRE"` y `sudo tail -1 /var/log/remoto.log`. En tu VM (UTM) la IP host-only es otra: usar la tuya.
