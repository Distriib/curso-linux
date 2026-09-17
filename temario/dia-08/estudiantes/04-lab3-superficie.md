# Lab 4.3 — Superficie: qué escucha, qué se cierra, parches solos

Vamos a mirar el servidor como lo ve un atacante, cerrar en el firewall lo que nadie pidió, y dejar las actualizaciones de seguridad en automático.

---

## Parte 1 — Qué corre y qué escucha

**¿Cuántas puertas tiene abiertas el servidor?**

```bash
systemctl list-units --type=service --state=running --no-pager
sudo ss -tulpn
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
  auditd.service        loaded active running Security Auditing Service
  chronyd.service       loaded active running NTP client/server
  firewalld.service     loaded active running firewalld - dynamic firewall daemon
  httpd.service         loaded active running The Apache HTTP Server
  sshd.service          loaded active running OpenSSH server daemon
  ...
udp   UNCONN 0  0    127.0.0.1:323    0.0.0.0:*  users:(("chronyd",...))
tcp   LISTEN 0  128    0.0.0.0:22     0.0.0.0:*  users:(("sshd",...))
tcp   LISTEN 0  511          *:80           *:*  users:(("httpd",...))
tcp   LISTEN 0  511          *:82           *:*  users:(("httpd",...))
```
Cada `LISTEN` en `0.0.0.0` o `*` es un servicio expuesto a la red. `chronyd` escucha solo en `127.0.0.1`: bien.

---

## Parte 2 — Cerrar lo que nadie pidió

**`cockpit` está abierto en el firewall de fábrica. ¿Alguien lo pidió?**

```bash
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --permanent --zone=internal --remove-service=cockpit
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
success
success
success
dhcpv6-client http ssh
```

---

## Parte 3 — Parches de seguridad solos

**¿Cómo hago que los parches de seguridad se apliquen sin que nadie se acuerde?**

```bash
sudo vim /etc/dnf/automatic.conf
```
Dos cambios: buscar `apply_updates` (`/apply_updates` `Enter`) y dejar `apply_updates = yes`; buscar `upgrade_type` (`/upgrade_type` `Enter`) y dejar `upgrade_type = security`. Guardar (`Esc` `:wq`).

```bash
grep apply_updates /etc/dnf/automatic.conf
grep upgrade_type /etc/dnf/automatic.conf
sudo systemctl enable --now dnf-automatic.timer
systemctl list-timers dnf-automatic.timer --no-pager
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
apply_updates = yes
upgrade_type = security
NEXT                        LEFT     LAST PASSED UNIT                ACTIVATES
... 06:..:.. ...            ..h left -    -      dnf-automatic.timer dnf-automatic.service
```
`security` = solo parches de seguridad, no todo. Corre una vez al día. No reinicia el servidor.

---

## Parte 4 — Lo que ya queda registrado

**¿Qué sabe el servidor de lo que hicimos hoy?**

```bash
sudo grep COMMAND /var/log/secure | tail -3
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** tres líneas con `student : TTY=pts/0 ; PWD=... ; USER=root ; COMMAND=/usr/bin/...`. Cada `sudo` del día: quién, desde dónde, qué comando.
