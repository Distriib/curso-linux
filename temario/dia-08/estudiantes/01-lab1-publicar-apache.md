# Lab 1.1 — Publicar Apache a través del firewall

Vamos a dejar el portal institucional visible desde tu computadora, abriendo el firewall bien (permanente), y a ver con las manos qué pasa cuando se abre mal.

| Dato | Valor |
|---|---|
| Servidor web | `httpd` (Apache), puerto 80 |
| Desde tu computadora, ruta NAT | `http://localhost:8080` |
| Desde tu computadora, ruta host-only | `http://192.168.56.10` (si tu IP host-only es otra, usá la tuya) |

> **Si `localhost:8080` no carga en ningún momento del día**, es que a tu VM le falta la regla de reenvío de puertos `8080 → 80` (la de SSH, `2222 → 22`, sí la tenés). Con la VM **apagada**, en VirtualBox: Configuración → Red → Adaptador 1 (NAT) → Avanzado → Reenvío de puertos → **+** → Host `8080`, Invitado `80`, TCP. En UTM: Configuración → Red → Reenvío de puertos → **+**, lo mismo.
>
> No es urgente: la ruta host-only (`http://192.168.56.10`) no necesita esa regla y sirve para todos los labs de hoy. Donde diga `localhost:8080`, usá la tuya.

---

## Parte 1 — Instalar y arrancar

**¿Qué hace falta para que el servidor tenga una página?**

```bash
sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic
sudo systemctl enable --now httpd
echo "<h1>Portal institucional - rhel01</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

(El `dnf` tarda un par de minutos.)

**Comprobar:**
```
<h1>Portal institucional - rhel01</h1>
```
La última línea: Apache responde **desde adentro** de la VM.

---

## Parte 2 — Desde afuera

**¿Y desde tu computadora?**

En el navegador de **tu computadora** (no en la VM): `http://localhost:8080` y `http://192.168.56.10`.

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

**Comprobar:** una línea con `tcp dport 80 ... accept`, y varios cientos de líneas en total. Cada `--add-service` termina siendo una línea de nftables. No se edita a mano: el próximo `--reload` lo pisa.

---

# Solución — todos los comandos

```bash
# Parte 1 — instalar y arrancar
sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic
sudo systemctl enable --now httpd
echo "<h1>Portal institucional - rhel01</h1>" | sudo tee /var/www/html/index.html
curl http://localhost

# Parte 2 — desde el navegador de tu computadora: localhost:8080 y 192.168.56.10
#            (las dos fallan: Apache anda, el firewall no deja entrar)

# Parte 3 — el estado del firewall
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all

# Parte 4 — los servicios predefinidos
sudo firewall-cmd --get-services | wc -w
sudo firewall-cmd --info-service=http
cat /usr/lib/firewalld/services/http.xml
ls /etc/firewalld/services/

# Parte 5 — abrirlo MAL: solo en runtime
sudo firewall-cmd --add-service=http
sudo firewall-cmd --list-services       # aparece http, y el navegador carga
sudo firewall-cmd --reload
sudo firewall-cmd --list-services       # http desapareció

# Parte 6 — abrirlo BIEN: permanente y recargado
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --list-services       # todavía sin http: --permanent no toca lo que corre
sudo firewall-cmd --reload
sudo firewall-cmd --list-services       # ahora sí
sudo firewall-cmd --query-service=http

# Parte 7 — un puerto suelto y --runtime-to-permanent
sudo firewall-cmd --add-port=7070/tcp
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --list-ports    # vacío: en disco todavía no está
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --permanent --list-ports    # ahora sí
sudo firewall-cmd --permanent --remove-port=7070/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports                # vacío otra vez

# Parte 8 — lo que quedó en el kernel
sudo nft list ruleset | grep "dport 80"
sudo nft list ruleset | wc -l
```

**La regla de oro:** `--permanent` escribe en disco pero no aplica nada; `--reload` aplica lo de disco y **borra** lo que no se guardó. Si algo "no funciona", faltó el `--reload`; si algo "dejó de funcionar" tras reiniciar, faltó el `--permanent`.
