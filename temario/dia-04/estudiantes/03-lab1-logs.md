# Lab 1 — Logs: `logger`, regla propia, accesos fallidos y `journalctl`

Vamos a mandar los mensajes de `monitor` a su propio archivo, generar intentos de acceso fallidos de verdad, y encontrarlos en los tres lugares donde quedan.

---

## Parte 1 — Las reglas que hay, y dónde cae `monitor` hoy

**¿A qué archivo van hoy los mensajes de `monitor`?**

```bash
grep /var/log /etc/rsyslog.conf
ls /etc/rsyslog.d/
sudo grep monitor /var/log/messages | tail -2
```

Foto.

**Comprobar:**
```
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
authpriv.*                                              /var/log/secure
mail.*                                                  -/var/log/maillog
cron.*                                                  /var/log/cron
uucp,news.crit                                          /var/log/spooler
local7.*                                                /var/log/boot.log
Sep  4 11:00:32 rhel01 monitor[5290]:  11:00:32 up 1:58,  1 user,  load average: 0.05, 0.09, 0.10
```
`/etc/rsyslog.d/` está vacío. `monitor` cae en `messages` porque `local0.info` cumple `*.info`.

---

## Parte 2 — Regla propia para `local0`

**¿Cómo hago que `local0` vaya a su propio archivo y no se duplique en `messages`?**

```bash
sudo vim /etc/rsyslog.d/monitor.conf
```

`i`, pegar, `Esc`, `:wq`:

```
local0.*    /var/log/monitor.log
& stop
```

```bash
sudo rsyslogd -N1
sudo systemctl restart rsyslog
logger -p local0.notice -t prueba "hola desde logger"
sudo tail -2 /var/log/monitor.log
sudo tail -1 /var/log/messages
```

Foto.

**Comprobar:**
```
rsyslogd: version 8.2102.0-..., config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
Sep  4 11:02:15 rhel01 monitor[5310]:  11:02:15 up 2:00,  1 user,  load average: ...
Sep  4 11:02:31 rhel01 prueba[5480]: hola desde logger
```
La última línea de `messages` **no** es la de `prueba`: `& stop` la frenó. Sin esa línea saldría en los dos archivos.

---

## Parte 3 — Generar intentos de acceso fallidos

Tres intentos de entrar por SSH a la propia VM con un usuario que no existe. Escribir cualquier contraseña las tres veces. A la pregunta de la huella, `yes`.

```bash
ssh intruso@localhost
```

Foto.

**Comprobar:**
```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
intruso@localhost's password:
Permission denied, please try again.
intruso@localhost's password:
Permission denied, please try again.
intruso@localhost's password:
intruso@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
```

---

## Parte 4 — El rastro, en tres lugares

**¿Dónde quedó registrado el intento?**

```bash
sudo grep 'Failed password' /var/log/secure | tail -3
sudo lastb | head -4
sudo journalctl -u sshd --since "10 min ago" --no-pager | grep Failed
```

Foto.

**Comprobar:**
```
Sep  4 11:05:02 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
Sep  4 11:05:05 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
Sep  4 11:05:08 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
intruso  ssh:notty    ::1              Thu Sep  4 11:05 - 11:05  (00:00)
intruso  ssh:notty    ::1              Thu Sep  4 11:05 - 11:05  (00:00)
intruso  ssh:notty    ::1              Thu Sep  4 11:05 - 11:05  (00:00)
Sep 04 11:05:02 rhel01 sshd[5510]: Failed password for invalid user intruso from ::1 port 47122 ssh2
...
```
La misma información en `secure` (texto), en `lastb` (binario) y en el journal (filtrado por unidad y tiempo). `::1` es "esta misma máquina".

---

## Parte 5 — Filtros de `journalctl`

**¿Qué falló en este arranque? ¿Y en el anterior? ¿Quién usó `sudo` hoy?**

```bash
sudo journalctl -b -p err --no-pager | tail -5
sudo journalctl -b -1 -n 3 --no-pager
sudo journalctl --since today -t sudo -n 3 --no-pager
```

Foto.

**Comprobar:**
```
(errores de este arranque; en una VM sana, casi nada)
Specifying boot ID or boot offset has no effect, no persistent journal was found.
Sep 04 10:14:02 rhel01 sudo[4300]:  student : TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/systemctl enable httpd
...
```
Si `-b -1` dice `no persistent journal was found`, el journal **se borra en cada reinicio**: lo arreglamos en el lab que sigue. Si en cambio muestra líneas de otro arranque, ya es persistente.
