# Lab 1.1 — Publicar Apache a través del firewall

Vamos a dejar el portal institucional visible desde tu computadora, abriendo el firewall bien (permanente), y a ver con las manos qué pasa cuando se abre mal.

| Dato | Valor |
|---|---|
| Servidor web | `httpd` (Apache), puerto 80 |
| Desde tu computadora, ruta NAT | `http://localhost:8080` |
| Desde tu computadora, ruta host-only | `http://192.168.56.10` (si tu IP host-only es otra, usá la tuya) |

---

## Parte 1 — Instalar y arrancar

**¿Qué hace falta para que el servidor tenga una página?**

```bash
sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic
sudo systemctl enable --now httpd
echo "<h1>Portal institucional - rhel01</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

Ahora ustedes: lo mismo en su VM (el `dnf` tarda un par de minutos). Foto.

**Comprobar:**
```
<h1>Portal institucional - rhel01</h1>
```
La última línea: Apache responde **desde adentro** de la VM.

---

## Parte 2 — Desde afuera

**¿Y desde tu computadora?**

En el navegador de **tu computadora** (no en la VM): `http://localhost:8080` y `http://192.168.56.10`.

Ahora ustedes: lo mismo. Foto.

**Comprobar:** ninguna de las dos carga (el navegador da error). Apache funciona; el firewall de la VM no deja entrar.

---

## Parte 3 — El estado del portero

**¿Qué está dejando pasar el firewall ahora?**

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
running
public
public
  interfaces: enp0s3 enp0s8
public (active)
  target: default
  ...
  services: cockpit dhcpv6-client ssh
  ports:
  ...
```
Las dos interfaces están en `public`; `http` no está en `services`.

---

## Parte 4 — Los servicios que ya vienen definidos

**¿Qué es "el servicio http" para el firewall?**

```bash
sudo firewall-cmd --get-services | wc -w
sudo firewall-cmd --info-service=http
cat /usr/lib/firewalld/services/http.xml
ls /etc/firewalld/services/
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** un número cerca de 190; `http` tiene `ports: 80/tcp`; el XML dice `<port protocol="tcp" port="80"/>`; la carpeta de `/etc` está vacía.

---

## Parte 5 — Abrirlo mal: solo en runtime

**¿Qué pasa si abro http sin `--permanent` y después recargo?**

```bash
sudo firewall-cmd --add-service=http
sudo firewall-cmd --list-services
```
En el navegador de tu computadora, `http://localhost:8080` ahora **carga**. Después:

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
success
cockpit dhcpv6-client http ssh
success
cockpit dhcpv6-client ssh
```
`http` desapareció con el `--reload`. Lo que no es `--permanent` muere con el reload y con el reinicio.

---

## Parte 6 — Abrirlo bien: permanente y recargado

**¿Cómo lo dejo abierto para siempre?**

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --list-services
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --query-service=http
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
success
cockpit dhcpv6-client ssh
success
cockpit dhcpv6-client http ssh
yes
```
La segunda línea **todavía no** tiene `http`: `--permanent` escribe en disco y no toca lo que corre. Después del `--reload`, sí. En tu computadora, `http://localhost:8080` carga, y va a seguir cargando después de reiniciar.

---

## Parte 7 — Un puerto suelto y `--runtime-to-permanent`

**¿Y si no hay un servicio predefinido para lo que quiero abrir?**

```bash
sudo firewall-cmd --add-port=7070/tcp
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --list-ports
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --permanent --list-ports
```

Y cerrarlo, porque hoy no lo usamos:
```bash
sudo firewall-cmd --permanent --remove-port=7070/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
success
7070/tcp
                  (línea vacía: en disco todavía no estaba)
success
7070/tcp
success
success
                  (línea vacía: cerrado)
```

---

## Parte 8 — Debajo de la alfombra

**¿Qué escribió firewalld en el kernel?**

```bash
sudo nft list ruleset | grep "dport 80"
sudo nft list ruleset | wc -l
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** una línea con `tcp dport 80 ... accept`, y varios cientos de líneas en total. Cada `--add-service` termina siendo una línea de nftables. No se edita a mano: el próximo `--reload` lo pisa.
