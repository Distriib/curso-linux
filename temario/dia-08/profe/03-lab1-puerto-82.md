# Lab — Apache en el puerto 82 (comandos)

Vos primero, ellos después, foto. Apache tiene que estar `active` en el 80 y sin `/etc/httpd/conf.d/puerto.conf` (Lab 2.1, Parte 3).

## Parte 1 — Preparar el traductor y ver qué puertos conoce SELinux
```bash
sudo service auditd restart
sudo semanage port -l | grep -w http_port_t
sudo semanage port -l | grep 8080
```
`service` es el comando anterior a systemd; en RHEL 9 sigue existiendo para casos especiales. auditd no se deja reiniciar con `systemctl restart` (la unidad lo prohíbe a propósito: `Operation refused`), así que se usa `service`. El reinicio hace que auditd cargue el complemento que le pasa las denegaciones a setroubleshoot; sin esto, en el Lab 3.4 `journalctl -t setroubleshoot` sale vacío. Qué decir: "esto es plomería: auditd tiene que enterarse de que instalamos el traductor."
Qué decir del 8080: "ya tiene dueño, `http_cache_port_t`, y a Apache la política lo deja usarlo igual. Por eso el 8080 'funciona solo' en los tutoriales. El 82 no tiene dueño: falla. Y eso es lo que queremos aprender a arreglar."

## Parte 2 — Reproducir el fallo y leer el AVC
```bash
echo "Listen 82" | sudo tee /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
sudo ausearch -m AVC -ts recent | tail -1
```
Leer el AVC con la tabla: `tclass=tcp_socket`, `name_bind`, `src=82`, `tcontext=...reserved_port_t` → fila 3 → `semanage port -a`.

## Parte 3 — Registrar el puerto
```bash
sudo semanage port -a -t http_port_t -p tcp 82
sudo semanage port -l | grep -w http_port_t
sudo systemctl restart httpd
curl http://localhost:82
```
Ahora Apache escucha en el 80 **y** en el 82 (el `Listen 80` de `httpd.conf` sigue).

## Parte 4 — Lo tuyo, y un puerto que ya tiene dueño
```bash
sudo semanage port -l -C
sudo semanage port -a -t http_port_t -p tcp 8080
```
El segundo falla a propósito con `ValueError: Port tcp/8080 already defined`. Qué decir: "para eso existe `-m`. No lo hacemos con el 8080 porque no lo necesitamos; lo van a necesitar en el reto con otro puerto." Para deshacer el 82 sería `sudo semanage port -d -t http_port_t -p tcp 82`: no ejecutarlo, lo usamos el resto del día.

## Parte 5 — Abrirlo en el firewall, en las dos zonas
```bash
sudo firewall-cmd --permanent --add-port=82/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=82/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
```
Que prueben `http://192.168.56.10:82` en el navegador (ruta host-only, zona `internal`). Por la ruta NAT no hay port forwarding para el 82: no probar `localhost:82`, no va a cargar y confunde.
