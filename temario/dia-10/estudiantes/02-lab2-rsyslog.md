# Lab 2.2 — rsyslog: regla propia y colector por red

Vamos a escribir una regla para la aplicación institucional y a convertir la VM en **servidor de logs** que recibe por TCP, probándolo desde ella misma.

| Qué | Valor |
|---|---|
| Facilidad de la aplicación | `local5` |
| Archivo propio | `/var/log/pgn-app.log` |
| Puerto del colector | `514/tcp` |
| Archivo de lo recibido | `/var/log/remoto.log` |

---

## Parte 1 — Cómo está armado rsyslog

**¿Dónde están las reglas que mandan `secure`, `cron` y `messages` a sus archivos?**

```bash
grep module /etc/rsyslog.conf
grep include /etc/rsyslog.conf
grep /var/log /etc/rsyslog.conf
ls /etc/rsyslog.d/
sudo rsyslogd -N1
```

**Comprobar:**
```
module(load="imuxsock" ...
module(load="imjournal" ...
include(file="/etc/rsyslog.d/*.conf" mode="optional")
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
authpriv.*                                              /var/log/secure
mail.*                                                  -/var/log/maillog
cron.*                                                  /var/log/cron
local7.*                                                /var/log/boot.log
monitor.conf
rsyslogd: End of config validation run. Bye.
```
`imjournal` = rsyslog lee del journal. La línea `include` está **antes** de las reglas: lo nuestro se evalúa primero.

---

## Parte 2 — Regla propia para `local5`

**¿Cómo mando los mensajes de mi aplicación a su propio archivo, sin que se dupliquen en `messages`?**

```bash
sudo vim /etc/rsyslog.d/pgn-app.conf
```
Contenido:
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

Ahora ustedes: manden un tercer mensaje con `-p local5.warning` y su nombre en el texto, y confirmen en qué archivo cayó. Foto.

**Comprobar:**
```
rsyslogd: End of config validation run. Bye.
Sep 16 ... rhel01 portal-pgn[NNNN]: Usuario ana consulto expediente 2026-001
Sep 16 ... rhel01 portal-pgn[NNNN]: Fallo de conexion a la base de datos
Sep 16 ... rhel01 portal-pgn[NNNN]: Fallo de conexion a la base de datos
```
`pgn-app.log` tiene los dos; `messages` solo el `err`. El `warning` de ustedes va solo a `pgn-app.log`.

---

## Parte 3 — La VM como servidor de logs

**¿Cómo recibo los logs de otros servidores por red?**

```bash
sudo vim /etc/rsyslog.d/remoto.conf
```
Contenido:
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

**Comprobar:**
```
rsyslogd: End of config validation run. Bye.
syslogd_port_t                 tcp      601, 20514
syslogd_port_t                 udp      514, 601, 20514
syslogd_port_t                 tcp      514, 601, 20514
syslogd_port_t                 udp      514, 601, 20514
LISTEN 0  25  0.0.0.0:514  0.0.0.0:*  users:(("rsyslogd",pid=...
```
Antes del `-m`, 514/tcp no era de syslog para SELinux (era de `rsh`); después sí. Sin ese paso, SELinux le prohíbe a rsyslog escuchar ahí.

---

## Parte 4 — Abrir el puerto en el firewall, en las dos zonas

```bash
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=514/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
```

**Comprobar:** `success` tres veces, y `514/tcp` en las dos listas.

---

## Parte 5 — Probar el colector

**¿Cómo pruebo que el colector recibe, sin tener otro servidor?**

```bash
logger -n 127.0.0.1 -P 514 -T -t prueba-directa "Mensaje enviado por TCP al colector"
sudo tail -2 /var/log/remoto.log
```

Ahora ustedes: manden otro mensaje con su nombre a la IP host-only de su VM (`192.168.56.10`) y confírmenlo en `remoto.log`. Foto.

**Comprobar:**
```
Sep 16 ... rhel01 prueba-directa Mensaje enviado por TCP al colector
```
`logger -n` manda directo al puerto; `-T` es TCP (sin `-T` usaría UDP, y el colector no escucha UDP). Esto es lo que ve el colector institucional con decenas de servidores. Para que un servidor **envíe todo** a un colector, en el cliente la regla es una sola línea: `*.*  @@IP:514`.
