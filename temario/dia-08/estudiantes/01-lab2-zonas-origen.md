# Lab 1.2 — La red de administración en su zona, y un Apache que no arranca

Vamos a decirle al firewall "lo que venga de la red host-only es de administración" (zona `internal`), a ver por qué eso rompe la web, y a arreglarlo. Al final, un encargo que deja a Apache caído a propósito.

| Dato | Valor |
|---|---|
| Red de administración (host-only) | `192.168.56.0/24` (si la tuya es otra, usá la tuya en todos los comandos) |
| Ruta NAT desde tu computadora | `http://localhost:8080` → entra por `public` |
| Ruta host-only desde tu computadora | `http://192.168.56.10` → hoy pasa a `internal` |

---

## Parte 1 — Conocer las zonas

**¿Qué zonas existen y en qué se diferencian?**

```bash
sudo firewall-cmd --get-zones
sudo firewall-cmd --info-zone=drop | head -3
sudo firewall-cmd --info-zone=trusted | head -3
sudo firewall-cmd --info-zone=public | head -3
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
block dmz drop external home internal nm-shared public trusted work
drop
  target: DROP
  icmp-block-inversion: no
trusted
  target: ACCEPT
  icmp-block-inversion: no
public (active)
  target: default
  icmp-block-inversion: no
```
`DROP` tira todo sin avisar; `ACCEPT` deja pasar todo; `default` = solo lo listado.

---

## Parte 2 — La red host-only es de administración

**¿Cómo le digo al firewall que lo que venga de 192.168.56.0/24 se evalúe en `internal`?**

```bash
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=internal --list-all
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
success
success
internal
  sources: 192.168.56.0/24
public
  interfaces: enp0s3 enp0s8
internal (active)
  target: default
  ...
  sources: 192.168.56.0/24
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports:
  ...
```
Ahora hay dos zonas activas. `internal` no tiene `http`.

---

## Parte 3 — Probar las dos rutas

**¿Cuál de las dos rutas va a cargar?**

En el navegador de tu computadora: `http://localhost:8080` y `http://192.168.56.10`.

Ahora ustedes: lo mismo. Foto.

**Comprobar:** la ruta NAT carga; la ruta host-only **dejó de cargar**, aunque `http` sigue abierto en `public`. Lo que viene de `192.168.56.x` ya no se evalúa en `public` (por la interfaz) sino en `internal` (por el origen), y en `internal` no está `http`. **El origen manda.**

---

## Parte 4 — Corregir: http también en internal

**¿Qué falta para que la ruta host-only vuelva a funcionar?**

```bash
sudo firewall-cmd --permanent --zone=internal --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --zone=internal --list-services
```

Ahora ustedes: lo mismo, y probar de nuevo las dos rutas en el navegador. Foto.

**Comprobar:**
```
success
success
cockpit dhcpv6-client http mdns samba-client ssh
```
Las dos rutas cargan. Regla de la casa desde hoy: **lo que se publica se abre en las dos zonas.**

---

## Parte 5 — El encargo: Apache también en el puerto 82

**El proveedor del portal pide que Apache responda también en el 82. ¿Qué puede salir mal?**

```bash
echo "Listen 82" | sudo tee /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
systemctl status httpd --no-pager -l
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Job for httpd.service failed because the control process exited with error code.
...
     Active: failed (Result: exit-code)
...
httpd[...]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:82
httpd[...]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:82
httpd[...]: no listening sockets available, shutting down
```
`Permission denied`... a **root** (Apache arranca como root). Con los permisos `rwx`, root puede abrir cualquier puerto. Alguien más está diciendo que no. Lo dejamos así, roto, para el siguiente bloque.
