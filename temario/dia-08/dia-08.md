# Día 08 — Seguridad: firewalld, SELinux y hardening básico
> Al terminar, el participante publica un servicio a través de firewalld con reglas por zona y por origen, diagnostica y corrige denegaciones de SELinux (contextos, puertos, booleanos) **sin apagarlo**, y aplica un hardening básico y verificable a SSH, contraseñas, actualizaciones y superficie de ataque del servidor.

**Ficha técnica cubierta:**
- RH124 M8: Firewall básico, SELinux introducción, Buenas prácticas
- RH134 M5: SELinux avanzado, Firewalld, Hardening básico

**Requisitos previos:**
- Snapshot `dia07-fin` tomado. Si algo quedó roto del Día 07, restaurarlo antes de empezar.
- VM `rhel01` con dos adaptadores: NAT (`enp0s3` en VirtualBox, `enp0s1` aprox. en UTM) con port forwarding 2222→22 y 8080→80, y Host-only con IP estática `192.168.56.10/24` configurada el Día 05. Verificar con `ip -4 addr` y `nmcli device`.
- Llave SSH del equipo del participante ya copiada a la VM (Día 05): `ssh -p 2222 student@localhost` entra sin pedir contraseña. **Sin esto no se puede hacer el Lab 4.1 con seguridad.**
- Suscripción activa: `sudo dnf repolist` muestra los repositorios BaseOS y AppStream.
- Si el Día 07 dejó como tarea descargar paquetes, mejor; si no, se instalan durante el Bloque 1 (ver Notas para el instructor). Paquetes del día: `httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic openscap-scanner scap-security-guide`.
- Los usuarios de PanamaTech (`ana`, `carlos`, `pedro`) pueden existir de días anteriores; no interfieren.

---

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---|---|---|
| 0:00–0:05 | 5 min | Apertura | Repaso del Día 07; seguridad en capas y el "por qué" visto desde el ataque; plan del día |
| 0:05–0:20 | 15 min | Bloque 1 — Conceptos | firewalld: arquitectura sobre nftables, zonas, servicios predefinidos, runtime vs permanent |
| 0:20–0:45 | 25 min | Bloque 1 — Lab 1.1 | Publicar httpd (e instalar los paquetes del día): abrir/cerrar http, puertos, query, runtime-to-permanent, mirar debajo con nft |
| 0:45–1:05 | 20 min | Bloque 1 — Lab 1.2 | Zonas por origen (red host-only), "cable de vida" SSH (demo), rich rules con log, panic mode; Apache al puerto 82 falla |
| 1:05–1:20 | 15 min | Bloque 2 — Conceptos | SELinux: DAC vs MAC, modos, contextos, tipos, política targeted |
| 1:20–1:40 | 20 min | Bloque 2 — Labs 2.1 y 2.2 | Modos y contextos (probar permissive y volver); el clásico mv → 403 → restorecon |
| 1:40–1:55 | 15 min | Descanso | |
| 1:55–2:05 | 10 min | Bloque 3 — Conceptos | semanage: puertos, fcontext, booleanos; anatomía de un AVC; tabla de decisión |
| 2:05–2:50 | 45 min | Bloque 3 — Labs 3.1 a 3.4 | Apache en 82 con semanage port; DocumentRoot /web con fcontext; booleanos; AVC con sealert y permissive por dominio |
| 2:50–2:55 | 5 min | Bloque 4 — Conceptos | Hardening: superficie, autenticación, actualizaciones, auditoría |
| 2:55–3:30 | 35 min | Bloque 4 — Labs 4.1 a 4.4 | sshd endurecido y probado desde el host; pwquality, faillock y chage; superficie y dnf-automatic; OpenSCAP (demo) |
| 3:30–3:50 | 20 min | Reto individual | Ticket: "la web institucional no carga desde afuera y da 403 desde adentro" |
| 3:50–4:00 | 10 min | Cierre | Checklist de 15 puntos, cheatsheet, tarea, snapshot `dia08-fin` |

---

## Prioridad si falta tiempo

**Imprescindible**
- firewalld: `--list-all`, `--add-service`/`--add-port` con `--permanent` + `--reload`, `--get-active-zones`, zona por origen (`--add-source`). Lab 1.1 completo y los pasos 1–3 **y 7** del Lab 1.2 (el paso 7 —dejar Apache roto en el puerto 82— es el que alimenta los Labs 2.1 y 3.1: no se puede saltar).
- SELinux: `getenforce`/`setenforce`, `ls -Z`, `ps -eZ`, el clásico `mv` → 403 → `restorecon -v` (Lab 2.2).
- `semanage port -a -t http_port_t -p tcp 82` (Lab 3.1) y `semanage fcontext -a` + `restorecon -Rv` (Lab 3.2).
- Leer un AVC con `ausearch -m AVC -ts recent` y decidir con la tabla de decisión.
- Hardening de sshd con la secuencia segura (Lab 4.1).
- El Reto individual: integra las tres capas del día.

**Importante**
- Rich rules y logging (Lab 1.2 pasos 4–5), `nft list ruleset`.
- Booleanos con `setsebool -P` (Lab 3.3), `sealert -l` (Lab 3.4), `semanage permissive`.
- pwquality y faillock (Lab 4.2, pasos 1–6).
- `dnf-automatic` y auditoría de puertos con `ss -tulpn` (Lab 4.3).

**Si sobra tiempo (o demo del instructor / tarea)**
- `--change-interface` y `connection.zone` de NetworkManager.
- SSH en un segundo puerto con `semanage port -a -t ssh_port_t`.
- `audit2allow` (solo lectura de lo que generaría).
- Caducidad con `chage` (Lab 4.2, paso 7): ya se practicó en el Día 03; si el Bloque 4 va apretado, el instructor lo muestra en pantalla y queda como tarea.
- OpenSCAP (Lab 4.4): el instructor lo muestra con un reporte ya generado.
- Menciones: auditd rules, fail2ban (EPEL), fapolicyd, `umask` 027, sudo con `log_output`.

---

## Bloque 1 — firewalld: el portero del servidor

### Conceptos (15 min)

**Por qué empezamos por el firewall (2 min de "sombrero negro").** Cuando un atacante escanea un servidor, lo primero que obtiene es la lista de puertos abiertos. Cada puerto abierto es una puerta que un servicio mantiene abierta las 24 horas; si ese servicio tiene una vulnerabilidad (y todos la tienen alguna vez), esa puerta es la entrada. El firewall del host no reemplaza al firewall perimetral de la institución: es la última barrera cuando el atacante ya está en la red interna, que es donde ocurren la mayoría de los movimientos laterales. En RHEL 9 recién instalado, la zona pública ya deja pasar `ssh`, `cockpit` (puerto 9090) y `dhcpv6-client`. Hoy vamos a decidir nosotros qué queda abierto.

**Arquitectura.** En RHEL 9 el filtrado real lo hace el kernel a través de **nftables** (sucesor de iptables). **firewalld** es el servicio que traduce reglas "humanas" (zonas, servicios, puertos) a reglas nftables, las mantiene y las recarga sin cortar conexiones. Se administra con `firewall-cmd`. Analogía: nftables es el mecanismo de la cerradura; firewalld es el portero que decide quién entra según una lista, y `firewall-cmd` es la forma de hablarle al portero.

```text
firewall-cmd  →  firewalld (daemon, D-Bus)  →  nftables (kernel)
                 /etc/firewalld/  (config del admin)
                 /usr/lib/firewalld/ (config por defecto: no se edita)
```

**Zonas.** Una zona es un nivel de confianza con un conjunto de reglas. El tráfico entrante se clasifica en una zona según **de dónde viene**: primero por la IP de origen (`sources`), después por la interfaz por la que entra (`interfaces`), y si nada coincide, la zona por defecto. Es decir: **el origen manda sobre la interfaz**. Zonas que hay que conocer:

| Zona | Comportamiento por defecto | Uso típico |
|---|---|---|
| `drop` | Descarta todo lo entrante sin responder. Solo salida. | Redes hostiles |
| `block` | Rechaza todo lo entrante con ICMP prohibited. | Igual, pero "educado" |
| `public` | **Zona por defecto.** Solo lo listado: ssh, cockpit, dhcpv6-client. | Interfaces expuestas |
| `external` | Como public, con masquerade (NAT). | Router |
| `internal` | Confía algo más: ssh, mdns, samba-client, dhcpv6-client, cockpit. | Red interna de administración |
| `trusted` | Acepta **todo**. | Solo para interfaces 100% confiables |

Asignar una interfaz a una zona significa "todo lo que entre por aquí se evalúa con estas reglas". Asignar un **origen** (una red) a una zona significa "todo lo que venga de estas IP se evalúa con estas reglas, entre por donde entre". Esto último es lo que usaremos para decir "solo la red de administración puede entrar a SSH".

**Servicios predefinidos.** Un servicio de firewalld es un archivo XML que agrupa puertos: `/usr/lib/firewalld/services/http.xml` dice "80/tcp". En RHEL 9 hay alrededor de 190 (el número exacto varía con la versión del paquete). Se listan con `firewall-cmd --get-services` y se inspeccionan con `--info-service=NOMBRE`. Si un servicio no existe, se abre el puerto directamente con `--add-port=8082/tcp`. Un servicio propio se define copiando un XML a `/etc/firewalld/services/`.

**Runtime vs permanent: la regla de oro.** Todo `firewall-cmd` sin `--permanent` cambia la configuración **en ejecución**, se aplica de inmediato y **se pierde** al recargar o reiniciar. Con `--permanent` se escribe en `/etc/firewalld/` pero **no se aplica** hasta `--reload`. Tres formas de trabajar:

1. Probar en runtime, y si funciona, `--runtime-to-permanent`.
2. Escribir con `--permanent` y aplicar con `--reload`.
3. Hacer el comando dos veces (con y sin `--permanent`). Funciona, pero es propenso a olvidos.

El error más común del examen y de la vida real: abrir con `--permanent`, no hacer `--reload`, y "no funciona"; o abrir sin `--permanent`, reiniciar el servidor, y "dejó de funcionar".

**Rich rules.** Cuando zonas y servicios no alcanzan (por ejemplo, "permitir SSH solo desde 192.168.56.0/24 y registrarlo en el log"), firewalld tiene un lenguaje de reglas enriquecidas: `rule family="ipv4" source address="..." service name="ssh" log prefix="..." accept`. Dentro de cada zona firewalld genera varias cadenas nftables en este orden: `_pre` → `_log` → `_deny` → `_allow` → `_post`. Es decir: **las rich rules que rechazan (`reject`/`drop`) se evalúan antes que los `services` y `ports` de la zona**, y las que aceptan conviven con ellos en la cadena `_allow`. Regla práctica: una rich rule sirve para excluir una IP concreta o para registrar; no sirve para "cerrar" algo que la zona ya abrió con `accept`.

**Panic mode.** `firewall-cmd --panic-on` descarta todo el tráfico, incluida la sesión SSH desde la que se ejecutó. Se menciona porque existe y porque aparece en la documentación; **no se ejecuta hoy**. Si alguien lo hace, se recupera desde la consola de la VM con `--panic-off`.

### Lab 1.1 — Publicar Apache a través del firewall (25 min)

- **Objetivo:** instalar httpd, comprobar que el firewall lo bloquea desde afuera, abrirlo bien (permanente) y entender la diferencia runtime/permanent con las manos.

1. Instalar y arrancar Apache con una página mínima. En el mismo `dnf` se instalan las herramientas de SELinux que se usarán en el Bloque 3 y `dnf-automatic`, que se configura en el Bloque 4 (`setroubleshoot-server` arrastra bastantes dependencias y tarda 1–3 minutos: mejor ahora, mientras se explica, que a mitad del Bloque 3). Si algo ya estaba instalado, dnf lo omite y se continúa.

```bash
sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic
sudo systemctl enable --now httpd
echo "<h1>Portal institucional - rhel01</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

Salida esperada (última línea):
```text
<h1>Portal institucional - rhel01</h1>
```

2. Desde el **equipo propio** (no desde la VM): abrir en el navegador `http://localhost:8080` (llega por el port forwarding NAT) y `http://192.168.56.10` (llega por la red host-only). Alternativa en la terminal del host:

```bash
curl -m 5 http://192.168.56.10
```

Salida esperada:
```text
curl: (7) Failed to connect to 192.168.56.10 port 80 ... No route to host
```
(o `curl: (28) Connection timed out` por la ruta NAT). Qué observar: Apache funciona dentro, pero el firewall de la VM rechaza desde afuera. Este es el Ticket 1 del Día 10 en versión "sabemos qué pasa".

3. Ver el estado del portero.

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Salida esperada:
```text
running
public
public
  interfaces: enp0s3 enp0s8
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
Qué observar: las dos interfaces están en `public`; `http` no está en `services`. En UTM las interfaces se llamarán distinto (verificar con `nmcli device`).

4. Servicios predefinidos.

```bash
sudo firewall-cmd --get-services | tr ' ' '\n' | wc -l
sudo firewall-cmd --info-service=http
cat /usr/lib/firewalld/services/http.xml
ls /etc/firewalld/services/
```

Salida esperada (resumida):
```text
19x                                <- alrededor de 190 servicios predefinidos
http
  ports: 80/tcp
  protocols:
  source-ports:
  modules:
  destination:
  includes:
  helpers:
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>WWW (HTTP)</short>
  <description>HTTP is the protocol used to serve Web pages. ...</description>
  <port protocol="tcp" port="80"/>
</service>
```
Qué observar: `/etc/firewalld/services/` está vacío: ahí irían los servicios definidos por el administrador. Nunca se edita `/usr/lib/firewalld/`.

5. Abrir http **solo en runtime** y perderlo a propósito.

```bash
sudo firewall-cmd --add-service=http
sudo firewall-cmd --list-services
```

Salida esperada:
```text
success
cockpit dhcpv6-client http ssh
```

Ahora desde el host `http://192.168.56.10` y `http://localhost:8080` **cargan**. A continuación:

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Salida esperada:
```text
success
cockpit dhcpv6-client ssh
```
Qué observar: `http` desapareció. Lo que no es `--permanent` muere con el `--reload` (y con el reinicio del servidor).

6. Hacerlo bien: permanente y recargado.

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --list-services
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --query-service=http; echo "codigo de salida: $?"
```

Salida esperada:
```text
success
cockpit dhcpv6-client ssh          <- todavía no: permanent no toca runtime
success
cockpit dhcpv6-client http ssh
yes
codigo de salida: 0
```
Qué observar: `--query-service` devuelve `yes`/`no` y código 0/1: sirve para scripts (`if firewall-cmd --query-service=http; then ...`). Comprobar de nuevo desde el host: ahora carga y **seguirá cargando después de reiniciar**.

7. Puertos sueltos y `--runtime-to-permanent`.

```bash
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --list-ports
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --permanent --list-ports
```

Salida esperada:
```text
success
8080/tcp
                                   <- vacío: en permanent aún no estaba
success
8080/tcp
```

Qué observar: este `8080/tcp` es un puerto **de la VM** (nadie escucha ahí); no tiene relación con el `8080` del port forwarding, que es un puerto **del host** que VirtualBox/UTM reenvía al 80 de la VM. Y cerrarlo, porque hoy no lo usaremos:

```bash
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

Salida esperada: `success`, `success` y una línea vacía.

8. Mirar debajo de la alfombra: lo que firewalld escribió en nftables.

```bash
sudo nft list ruleset | grep -E 'dport (22|80) '
sudo nft list ruleset | grep -c .
```

Salida esperada (aproximada):
```text
		tcp dport 22 ct state { new, untracked } accept
		tcp dport 80 ct state { new, untracked } accept
3xx
```
Qué observar: cada `--add-service` termina siendo una línea nftables. No se edita nftables a mano en un servidor con firewalld: el siguiente `--reload` lo pisaría. (⚠️ Verificar en la VM antes de la clase: el formato exacto de la línea cambia entre versiones de firewalld/nftables; si el `grep` no devuelve nada, usar `sudo nft list ruleset | grep -n 'dport'` y mostrar lo que salga. Lo que se enseña es la idea, no el texto literal.)

- **Checkpoint:** pegar en el chat la salida de
```bash
sudo firewall-cmd --list-all | grep -E 'services|ports:'
```
Debe mostrar `services: cockpit dhcpv6-client http ssh` y `ports:` vacío.

### Lab 1.2 — Zonas por origen, rich rules y un Apache que se niega a arrancar (20 min)

- **Objetivo:** clasificar la red host-only como red de administración (zona `internal`), demostrar que el origen manda sobre la interfaz, registrar accesos SSH con una rich rule y provocar el fallo que nos lleva a SELinux.

1. Conocer las zonas.

```bash
sudo firewall-cmd --get-zones
sudo firewall-cmd --info-zone=drop | head -3
sudo firewall-cmd --info-zone=trusted | head -3
sudo firewall-cmd --list-all-zones | grep -E '^[a-z]|target'
```

Salida esperada (resumida):
```text
block dmz drop external home internal nm-shared public trusted work
drop
  target: DROP
  icmp-block-inversion: no
trusted
  target: ACCEPT
  icmp-block-inversion: no
block
  target: %%REJECT%%
...
public (active)
  target: default
...
```

2. La red host-only es nuestra red de administración: asignarla como **origen** a `internal`.

```bash
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=internal --list-all
```

Salida esperada:
```text
success
success
internal
  sources: 192.168.56.0/24
public
  interfaces: enp0s3 enp0s8
internal (active)
  target: default
  icmp-block-inversion: no
  interfaces:
  sources: 192.168.56.0/24
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports:
  ...
```

Ahora, desde el host, probar **las dos rutas** de la web:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -m 5 http://localhost:8080
curl -s -o /dev/null -w '%{http_code}\n' -m 5 http://192.168.56.10
```

Salida esperada:
```text
200
000
```
Qué observar: la ruta host-only **dejó de funcionar** aunque `http` sigue abierto en `public`. Los paquetes que vienen de 192.168.56.x ya no se evalúan en `public` (por interfaz) sino en `internal` (por origen), y en `internal` no está `http`. **El origen manda.** Esta es la fuente número uno de confusión con zonas.

3. Corregir: abrir http también en `internal`, y aprovechar para dejar claro que `internal` es la única con SSH garantizado hacia la red de administración.

```bash
sudo firewall-cmd --permanent --zone=internal --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --zone=internal --list-services
```

Salida esperada: `success`, `success`, `cockpit dhcpv6-client http mdns samba-client ssh`. Repetir el `curl` a `192.168.56.10` desde el host: `200`.

4. **Demostración controlada (runtime, reversible; la hace el instructor y los participantes la replican solo si ya tienen la sesión host-only abierta):** "solo la red de administración entra a SSH".
   - En el host, abrir una **segunda terminal** y conectarse por la ruta host-only: `ssh student@192.168.56.10`. Esta sesión es el cable de vida: se evalúa en `internal`, donde `ssh` seguirá abierto. **Confirmar que esta sesión funciona antes de tocar nada.** Si no se tiene red host-only, no ejecutar este paso: solo observar.
   - En **esa** sesión, quitar ssh de `public` **sin** `--permanent`:

```bash
sudo firewall-cmd --zone=public --remove-service=ssh
sudo firewall-cmd --list-services
```

Salida esperada: `success` y `cockpit dhcpv6-client http`.

   - En el host, en una tercera terminal, intentar la ruta NAT (el origen que ve la VM es la puerta NAT `10.0.2.2`, que cae en `public`):

```bash
ssh -p 2222 -o ConnectTimeout=5 student@localhost
```

Salida esperada: `ssh: connect to host localhost port 2222: Connection timed out`, `Connection refused` o `kex_exchange_identification: Connection closed by remote host` (depende de cómo VirtualBox/UTM propaguen el rechazo de la VM al host). La sesión host-only sigue viva. Qué observar: las sesiones SSH ya establecidas (incluida una que estuviera abierta por NAT) **no se cortan**, porque firewalld acepta primero `ct state established`. El peligro es silencioso: se nota al reconectar. Por eso este cambio jamás se hace `--permanent` sin una consola de respaldo.

   - **Salida de emergencia:** si alguien ejecutó el `--remove-service=ssh` sin tener la sesión host-only y pierde el acceso al reconectar, se entra por la **consola de la VM** (VirtualBox/UTM, usuario `student`) y se ejecuta `sudo firewall-cmd --reload`. Como el cambio nunca fue `--permanent`, el `--reload` lo deshace por completo.

   - Restaurar: como el cambio fue runtime, basta recargar.

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Salida esperada: `success` y `cockpit dhcpv6-client http ssh`.

5. Rich rule con registro: quién entra a SSH desde la red de administración.

```bash
sudo firewall-cmd --permanent --zone=internal --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" service name="ssh" log prefix="SSH-ADMIN " level="info" limit value="5/m" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --zone=internal --list-rich-rules
```

Salida esperada:
```text
success
success
rule family="ipv4" source address="192.168.56.0/24" service name="ssh" log prefix="SSH-ADMIN " level="info" limit value="5/m" accept
```

Desde el host: `ssh student@192.168.56.10 exit`. En la VM:

```bash
sudo journalctl -k --since "2 min ago" | grep SSH-ADMIN | tail -2
```

Salida esperada (aproximada):
```text
sep 03 09:41:12 rhel01 kernel: SSH-ADMIN IN=enp0s8 OUT= MAC=... SRC=192.168.56.1 DST=192.168.56.10 ... PROTO=TCP SPT=51422 DPT=22 ... SYN ...
```
Qué observar: el log de nftables va al kernel, por eso `journalctl -k` (o `dmesg`). Ojo con la lectura fácil: esta rich rule **no restringe nada**, porque `ssh` sigue en la lista de `services` de `internal`; lo único que agrega es el registro. Para que "solo la red de administración entre a SSH" hay que **quitar `ssh` de las demás zonas** (eso es lo que se demostró en el paso 4). Una rich rule que **rechaza** un origen concreto se escribe igual con `reject` o `drop` en vez de `accept` —y esas sí se evalúan antes que los `services` de la zona—, y se quita con `--remove-rich-rule='...'` usando exactamente el mismo texto. El `limit value="5/m"` limita las **líneas de log**, no las conexiones.

6. Panic mode: solo mirar.

```bash
sudo firewall-cmd --query-panic
```

Salida esperada: `no`. Explicar `--panic-on`/`--panic-off` y **no ejecutarlo**.

7. Ahora el encargo del día: "el puerto 80 lo va a ocupar un proxy; Apache debe escuchar en el **82**".

```bash
sudo sed -i 's/^Listen 80$/Listen 82/' /etc/httpd/conf/httpd.conf
grep -n '^Listen' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
systemctl status httpd --no-pager -l | head -15
```

Salida esperada (resumida; el número de línea puede variar):
```text
47:Listen 82
Job for httpd.service failed because the control process exited with error code.
× httpd.service - The Apache HTTP Server
     Active: failed (Result: exit-code)
  ...
  rhel01 httpd[2210]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:82
  rhel01 httpd[2210]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:82
  rhel01 httpd[2210]: no listening sockets available, shutting down
```
Qué observar: `Permission denied`... a **root** (httpd arranca como root y luego baja privilegios). En el modelo clásico de permisos, root puede abrir cualquier puerto. Alguien más está diciendo que no. Ese alguien es SELinux, y lo dejamos así, roto, para el siguiente bloque.

- **Checkpoint:** pegar en el chat la salida de
```bash
sudo firewall-cmd --get-active-zones; systemctl is-active httpd
```
Debe mostrar `internal` con `sources: 192.168.56.0/24`, `public` con las interfaces, y `failed`.

---

## Bloque 2 — SELinux: introducción

### Conceptos (15 min)

**El problema que resuelve (con el sombrero negro puesto).** Los permisos `rwx` son **DAC** (Discretionary Access Control): el dueño del archivo decide, y root puede todo. Si un atacante explota Apache y consigue ejecutar código, ese código corre como el usuario `apache`… y lo primero que intentará es leer `/etc/passwd` y `/etc/shadow`, escribir una webshell en un directorio con permisos de escritura, abrir una conexión de salida hacia su servidor de comando y control, o leer los archivos de `/home`. Con solo DAC, varias de esas cosas funcionan. SELinux agrega **MAC** (Mandatory Access Control): una política, escrita por Red Hat y que el usuario no puede saltarse, dice qué puede hacer **cada proceso** según su **tipo**, sin importar quién sea el usuario. Un `httpd_t` solo puede leer `httpd_sys_content_t`, escuchar en `http_port_t` y poco más. El proceso comprometido sigue comprometido, pero encerrado en una habitación pequeña. Esto no es teoría: SELinux ha frenado exploits reales (Shellshock contra CGI, varios RCE en aplicaciones web) en servidores donde estaba en enforcing.

**Analogía.** DAC es el candado del casillero: el dueño decide a quién le da la llave. MAC es el reglamento del edificio: aunque se tenga la llave, el guardia no deja entrar a un piso que no corresponde a la credencial. Las dos comprobaciones ocurren; **ambas** deben aprobar.

**Modos.**
- `Enforcing`: la política se aplica y se registra lo denegado. Es el modo de producción y el del examen.
- `Permissive`: la política **no** se aplica pero **sí** se registra lo que habría denegado. Es la herramienta de diagnóstico: "si en permissive funciona, es SELinux".
- `Disabled`: no hay política cargada. En RHEL 9, poner `SELINUX=disabled` en `/etc/selinux/config` ya no deshabilita del todo; para eso hace falta el parámetro de kernel `selinux=0` (`grubby --update-kernel ALL --args selinux=0`). Se menciona para saber que existe; **deshabilitar SELinux es el pecado capital** del examen y de la administración seria. Después de volver a habilitarlo hay que reetiquetar todo el sistema (`touch /.autorelabel` y reiniciar), que tarda minutos.

`setenforce 0/1` cambia el modo hasta el próximo reinicio; `/etc/selinux/config` fija el modo de arranque.

**Contextos.** Todo archivo, proceso, puerto y usuario tiene una etiqueta de cuatro campos: `usuario:rol:tipo:nivel`, por ejemplo `system_u:object_r:httpd_sys_content_t:s0`. Con la política **targeted** (la de RHEL), lo que importa el 95 % del tiempo es el **tipo** (termina en `_t`). Se ve con `ls -Z` (archivos), `ps -eZ` (procesos), `id -Z` (mi sesión) y `semanage port -l` (puertos).

Tipos que hay que reconocer a la vista:

| Tipo | Qué es |
|---|---|
| `httpd_t` | El proceso Apache (dominio) |
| `httpd_sys_content_t` | Contenido web que Apache puede leer |
| `httpd_sys_rw_content_t` | Contenido web donde Apache puede escribir |
| `http_port_t` | Puertos donde Apache puede escuchar (80, 443, 8008…) |
| `ssh_port_t` | Puerto 22 |
| `admin_home_t` | `/root` y lo que se crea ahí |
| `user_home_t` / `user_home_dir_t` | Archivos y directorios en `/home/usuario` |
| `default_t` | Directorio nuevo en `/` sin regla: nadie confinado puede leerlo |
| `shadow_t` | `/etc/shadow`: casi ningún dominio puede leerlo |
| `unconfined_t` | Mi shell como usuario: sin restricciones de SELinux |

**De dónde sale la etiqueta de un archivo.** Un archivo nuevo **hereda el tipo del directorio** donde se crea (con excepciones definidas en la política). `cp` crea un archivo nuevo en el destino: hereda el contexto correcto. `mv` **mueve el mismo inodo**: conserva el contexto del origen. Por eso el clásico "lo copié a /var/www/html y funciona; lo moví y da 403". `restorecon` aplica la etiqueta que la política dice que **debería** tener esa ruta.

### Lab 2.1 — Modos, la prueba del permissive y contextos (10 min)

- **Objetivo:** confirmar con `setenforce 0` que el fallo de Apache es de SELinux, volver a enforcing, y leer contextos de archivos, procesos y sesión.

1. Estado actual.

```bash
getenforce
sestatus
```

Salida esperada:
```text
Enforcing
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

2. La prueba definitiva: permissive.

```bash
sudo setenforce 0
getenforce
sudo systemctl restart httpd
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82
sudo ausearch -m AVC -ts recent | grep name_bind | tail -1
```

Salida esperada:
```text
Permissive
200
type=AVC msg=audit(1788...): avc:  denied  { name_bind } for  pid=2350 comm="httpd" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=1
```
Qué observar: en permissive Apache arranca en el 82 → **confirmado: era SELinux**. Y la denegación queda registrada igual (`permissive=1`). Esto es lo que se hace en producción para diagnosticar: unos segundos en permissive, nunca días.

3. Volver a enforcing y dejar Apache en el 80 mientras aprendemos a arreglarlo bien (Bloque 3).

```bash
sudo setenforce 1
sudo systemctl restart httpd; systemctl is-active httpd
sudo sed -i 's/^Listen 82$/Listen 80/' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd; systemctl is-active httpd
grep -E '^SELINUX' /etc/selinux/config
```

Salida esperada:
```text
Job for httpd.service failed because the control process exited with error code. ...
failed
active
SELINUX=enforcing
SELINUXTYPE=targeted
```
Qué observar: en enforcing el 82 vuelve a fallar (primer `failed`); con `Listen 80` arranca (`active`). No se toca `/etc/selinux/config`: el modo de arranque sigue siendo `enforcing`.

4. Contextos de sesión, proceso y archivos.

```bash
id -Z
ps -eZ | grep httpd | head -2
ls -Z /var/www/html/
ls -Zd /root /home/student /etc/shadow /var/www/html /tmp
```

Salida esperada:
```text
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
system_u:system_r:httpd_t:s0     2410 ?        00:00:00 httpd
system_u:system_r:httpd_t:s0     2411 ?        00:00:00 httpd
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
system_u:object_r:admin_home_t:s0 /root
unconfined_u:object_r:user_home_dir_t:s0 /home/student
system_u:object_r:shadow_t:s0 /etc/shadow
system_u:object_r:httpd_sys_content_t:s0 /var/www/html
system_u:object_r:tmp_t:s0 /tmp
```
Qué observar: el primer campo (`system_u`/`unconfined_u`) puede variar según quién creó el archivo; lo que decide el acceso es el **tipo**. Mi shell es `unconfined_t`: SELinux no me limita a mí; limita a los servicios.

### Lab 2.2 — El clásico: mv, 403 y restorecon (10 min)

- **Objetivo:** reproducir el error más frecuente de SELinux en servidores web y corregirlo en un comando.

1. Crear la página como root en su home y **moverla**.

```bash
sudo -i
echo "<h1>Portal institucional - version 2</h1>" > /root/index.html
ls -Z /root/index.html
mv /root/index.html /var/www/html/index.html
exit
```

Salida esperada (del `ls -Z`):
```text
unconfined_u:object_r:admin_home_t:s0 /root/index.html
```

2. Comprobar el síntoma y los permisos "correctos".

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost
ls -l /var/www/html/index.html
ls -Z /var/www/html/index.html
```

Salida esperada:
```text
403
-rw-r--r--. 1 root root 42 sep  3 10:02 /var/www/html/index.html
unconfined_u:object_r:admin_home_t:s0 /var/www/html/index.html
```
Qué observar: permisos 644, dueño root, todo el mundo puede leer… y Apache da 403. El punto después de los permisos (`-rw-r--r--.`) indica que el archivo tiene contexto SELinux. El tipo es `admin_home_t`: la etiqueta viajó con el archivo.

3. Ver la denegación y corregir.

```bash
sudo ausearch -m AVC -ts recent | grep index.html | tail -1
sudo restorecon -v /var/www/html/index.html
curl http://localhost
```

Salida esperada:
```text
type=AVC msg=audit(...): avc:  denied  { getattr } for  pid=2411 comm="httpd" path="/var/www/html/index.html" dev="dm-0" ino=... scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file permissive=0
Relabeled /var/www/html/index.html from unconfined_u:object_r:admin_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
<h1>Portal institucional - version 2</h1>
```
Qué observar: la operación negada puede ser `{ getattr }` o `{ read }` según en qué punto tropiece Apache; lo que importa es `comm="httpd"`, `tcontext=...admin_home_t` y `tclass=file`. `restorecon` cambia solo el **tipo** (conserva `unconfined_u`), que es lo que decide el acceso.

4. Comparar `cp`, `cp -a` y `mv -Z`.

```bash
sudo -i
echo "<p>aviso</p>" > /root/aviso.html
cp /root/aviso.html /var/www/html/aviso-cp.html
cp -a /root/aviso.html /var/www/html/aviso-cpa.html
mv -Z /root/aviso.html /var/www/html/aviso-mvZ.html
ls -Z /var/www/html/aviso-*
rm -f /var/www/html/aviso-*
exit
```

Salida esperada:
```text
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/aviso-cp.html
unconfined_u:object_r:admin_home_t:s0 /var/www/html/aviso-cpa.html
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/aviso-mvZ.html
```
Qué observar: `cp` hereda del destino (bien); `cp -a` preserva **todo**, incluido el contexto (trampa frecuente en scripts de despliegue); `mv -Z` mueve y reetiqueta según el destino. Regla práctica: después de mover contenido a un directorio de servicio, `restorecon -Rv` siempre.

- **Checkpoint:** pegar en el chat la salida de
```bash
ls -Z /var/www/html/index.html; curl -s -o /dev/null -w '%{http_code}\n' http://localhost
```
Debe mostrar `httpd_sys_content_t` y `200`.

---

## Bloque 3 — SELinux avanzado: semanage y diagnóstico

### Conceptos (10 min)

Hasta aquí corregimos etiquetas que la política ya conocía. Ahora vamos a **modificar la política local** para casos legítimos: un puerto no estándar, un directorio propio, una funcionalidad opcional. La herramienta es `semanage` (paquete `policycoreutils-python-utils`); los cambios son persistentes y sobreviven a reinicios y reetiquetados.

**Puertos** (`semanage port`). Cada puerto tiene un tipo. Apache solo puede hacer `name_bind` en `http_port_t`. Si debe escuchar en el 82, se etiqueta el 82 como `http_port_t` con `-a` (agregar). Si el puerto **ya tiene otro tipo** (por ejemplo 8080 es `http_cache_port_t`), `-a` falla con "already defined" y se usa `-m` (modificar). `-d` borra una etiqueta agregada localmente. `-l -C` lista solo las personalizaciones locales.

**Contextos de archivo** (`semanage fcontext`). La política es una tabla de expresiones regulares → tipo (`/var/www(/.*)? → httpd_sys_content_t`). `restorecon` consulta esa tabla. Si creamos `/web`, no hay regla, así que `restorecon` le pondrá `default_t` (y Apache no lo leerá). La solución es **agregar la regla** (`semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"`) y **aplicarla** (`restorecon -Rv /web`). Dos pasos, siempre. `chcon` cambia la etiqueta directamente sin tocar la tabla: funciona hoy y se pierde con el próximo `restorecon` o reetiquetado; solo sirve para pruebas. `fixfiles` es `restorecon` para todo el sistema; `touch /.autorelabel` + reinicio reetiqueta todo en el arranque.

**Booleanos** (`getsebool`, `setsebool`, `semanage boolean`). Interruptores que activan partes opcionales de la política sin escribir reglas. Ejemplos: `httpd_can_network_connect` (Apache como proxy inverso hacia otro servidor), `httpd_enable_homedirs` (publicar `~/public_html`), `ftpd_anon_write` (lo veremos en el Día 09 con FTP). `setsebool -P` es persistente; sin `-P` se pierde al reiniciar.

**Anatomía de un AVC.** Cada denegación queda en `/var/log/audit/audit.log` como un registro `AVC`:

```text
avc: denied { getattr } for pid=2411 comm="httpd" path="/web/config.txt"
     scontext=system_u:system_r:httpd_t:s0  tcontext=unconfined_u:object_r:admin_home_t:s0
     tclass=file permissive=0
```

| Campo | Pregunta que responde |
|---|---|
| `{ getattr }` / `{ read }` / `{ name_bind }` | ¿qué operación se negó? |
| `comm=` | ¿qué programa? |
| `scontext=` (source) | ¿qué dominio lo intentó? → `httpd_t` |
| `tcontext=` (target) | ¿qué etiqueta tenía el objeto? → aquí está casi siempre la pista |
| `tclass=` | ¿qué tipo de objeto? `file`, `dir`, `tcp_socket` |
| `path=` / `name=` / `src=` | ¿cuál archivo o puerto? |

**Herramientas de lectura:** `ausearch -m AVC -ts recent` (crudo, siempre disponible), `journalctl -t setroubleshoot` y `sealert -l UUID` (traducción a lenguaje humano con sugerencia, requiere `setroubleshoot-server`), `sealert -a /var/log/audit/audit.log` (analiza el log completo sin depender del daemon).

**Tabla de decisión SELinux** (imprimir):

| Lo que dice el AVC | Diagnóstico | Acción |
|---|---|---|
| `tclass=file/dir`, `tcontext` con un tipo "ajeno" (`admin_home_t`, `user_home_t`, `tmp_t`) en un directorio **estándar** (`/var/www/html`) | Archivo movido con contexto equivocado | `restorecon -Rv /ruta` |
| Igual, pero el directorio es **propio** (`/web`, `/sitio`) y `tcontext=default_t` | No hay regla fcontext | `semanage fcontext -a -t TIPO "/ruta(/.*)?"` + `restorecon -Rv` |
| `tclass=tcp_socket`, `{ name_bind }`, `src=PUERTO` | Puerto sin etiqueta para ese servicio | `semanage port -a -t TIPO_port_t -p tcp PUERTO` (o `-m` si ya está definido) |
| `tclass=tcp_socket`, `{ name_connect }` | El servicio quiere **salir** hacia otro puerto | Booleano (`httpd_can_network_connect`, `..._db`) |
| `httpd_t` sobre `user_home_t` | Publicar homes | `setsebool -P httpd_enable_homedirs on` |
| `sealert` sugiere un booleano con confianza alta | Funcionalidad opcional | `setsebool -P nombre on` |
| Nada de lo anterior y es una aplicación propia | Falta política | `semanage permissive -a dominio_t` mientras se analiza; `audit2allow` solo con revisión línea por línea |
| "Funciona en permissive" | Confirmación de que es SELinux | Volver a enforcing **antes** de corregir con lo de arriba |

Lo que **no** está en la tabla: `setenforce 0` como solución, `SELINUX=disabled`, `chcon` en producción.

### Lab 3.1 — Apache en el puerto 82 con semanage port (10 min)

- **Objetivo:** resolver el fallo del Bloque 1 registrando el puerto, y abrirlo en el firewall: el ejercicio clásico del RHCSA.

1. Confirmar las herramientas (ya se instalaron en el Lab 1.1: dnf dirá "Nothing to do") y reiniciar auditd para que cargue el plugin de setroubleshoot.

```bash
sudo dnf install -y policycoreutils-python-utils setroubleshoot-server
sudo service auditd restart
```

Salida esperada: del `dnf`, `Package httpd-... is already installed.` → `Nothing to do.` → `Complete!`; del `service`, `Stopping logging: [  OK  ]` / `Redirecting start to /bin/systemctl start auditd.service`. Qué observar: `systemctl restart auditd` es rechazado a propósito (la unidad trae `RefuseManualStop=yes`); auditd se reinicia con `service`, que usa la acción heredada de `/usr/libexec/initscripts/legacy-actions/auditd/`. El reinicio hace que auditd cargue el plugin `sedispatch` (`/etc/audit/plugins.d/sedispatch.conf`, del paquete `setroubleshoot-plugins`), que es quien alimenta a setroubleshoot.

2. Qué puertos conoce SELinux para Apache.

```bash
sudo semanage port -l | grep -w http_port_t
sudo semanage port -l | grep -wE '82|8080|8082'
```

Salida esperada:
```text
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
us_cli_port_t                  tcp      8082, 8083
us_cli_port_t                  udp      8082, 8083
```
Qué observar: el 82 no aparece en ningún tipo (los puertos < 1024 sin etiqueta son `reserved_port_t`); el 8080 ya es `http_cache_port_t`; el 8082 pertenece a otro tipo. Por eso muchos tutoriales que usan 8080 "funcionan solos": la política deja a Apache usar `http_cache_port_t`. Nosotros usamos el 82 porque **sí** falla, que es lo que queremos aprender a arreglar.

3. Reproducir el fallo, leer el AVC y corregir.

```bash
sudo sed -i 's/^Listen 80$/Listen 82/' /etc/httpd/conf/httpd.conf
sudo systemctl restart httpd
sudo ausearch -m AVC -ts recent | grep -oE 'denied .*tclass=[a-z_]+' | tail -1
sudo semanage port -a -t http_port_t -p tcp 82
sudo semanage port -l | grep -w http_port_t
sudo systemctl restart httpd
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82
```

Salida esperada:
```text
Job for httpd.service failed ...
denied  { name_bind } for  pid=2520 comm="httpd" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket
http_port_t                    tcp      82, 80, 81, 443, 488, 8008, 8009, 8443, 9000
200
```

4. Ver la personalización local, y probar qué pasa con un puerto ya definido.

```bash
sudo semanage port -l -C
sudo semanage port -a -t http_port_t -p tcp 8080
```

Salida esperada:
```text
SELinux Port Type              Proto    Port Number

http_port_t                    tcp      82
ValueError: Port tcp/8080 already defined
```
Qué observar: para reasignar 8080 a `http_port_t` habría que usar `-m` (modificar). Para deshacer el 82 sería `semanage port -d -t http_port_t -p tcp 82` (no lo ejecutamos: lo necesitamos).

5. Abrir el puerto en el firewall, en **las dos zonas** que reciben tráfico.

```bash
sudo firewall-cmd --permanent --add-port=82/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=82/tcp
sudo firewall-cmd --reload
```

Salida esperada: tres `success`. Desde el host: `http://192.168.56.10:82` carga. (Quien no tenga red host-only: agregar en VirtualBox un port forwarding 18082→82 y abrir `http://localhost:18082`.)

- **Checkpoint:** pegar en el chat la salida de
```bash
sudo semanage port -l -C; curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82
```
Debe mostrar `http_port_t   tcp   82` y `200`.

### Lab 3.2 — DocumentRoot en /web con fcontext (15 min)

- **Objetivo:** servir la web desde un directorio propio, ver por qué `chcon` no basta, y dejar una regla persistente.

1. Crear el directorio y apuntar Apache a él.

```bash
sudo mkdir -p /web
echo "<h1>Portal institucional - servido desde /web</h1>" | sudo tee /web/index.html
ls -Zd /web; ls -Z /web
sudo sed -i 's#^DocumentRoot "/var/www/html"#DocumentRoot "/web"#' /etc/httpd/conf/httpd.conf
sudo sed -i 's#^<Directory "/var/www/html">#<Directory "/web">#' /etc/httpd/conf/httpd.conf
grep -nE '^DocumentRoot|^<Directory "/web"' /etc/httpd/conf/httpd.conf
sudo apachectl configtest
sudo systemctl restart httpd
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82
```

Salida esperada:
```text
unconfined_u:object_r:default_t:s0 /web
unconfined_u:object_r:default_t:s0 index.html
124:DocumentRoot "/web"
136:<Directory "/web">
Syntax OK
403
```
Qué observar (los números de línea son orientativos): se cambia también el bloque `<Directory>` porque ahí está el `Require all granted`; sin él Apache daría 403 **por su propia configuración**, no por SELinux, y confundiría el diagnóstico. El directorio nuevo nace como `default_t`, que es el tipo "no tengo regla para esto" y que ningún dominio confinado puede leer. `apachectl configtest` puede imprimir además el aviso `AH00558: Could not reliably determine the server's fully qualified domain name` — es ruido, no un error. ⚠️ Verificar en la VM antes de la clase: si `ls -Zd /web` mostrara otro tipo (por ejemplo `root_t`, heredado de `/`), ejecutar `sudo restorecon -v /web` para dejarlo en `default_t`; el resto del lab es idéntico.

2. Confirmar con el AVC.

```bash
sudo ausearch -m AVC -ts recent | grep default_t | tail -1
```

Salida esperada (puede ser sobre el directorio, `path="/web" ... tclass=dir`, o sobre el archivo):
```text
type=AVC ... avc:  denied  { getattr } for  pid=... comm="httpd" path="/web/index.html" ... scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```

3. La tentación: `chcon`. Funciona… hasta el próximo `restorecon`.

```bash
sudo chcon -R -t httpd_sys_content_t /web
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82
sudo restorecon -Rv /web
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82
```

Salida esperada:
```text
200
Relabeled /web from unconfined_u:object_r:httpd_sys_content_t:s0 to unconfined_u:object_r:default_t:s0
Relabeled /web/index.html from unconfined_u:object_r:httpd_sys_content_t:s0 to unconfined_u:object_r:default_t:s0
403
```
Qué observar: `restorecon` "deshizo el arreglo" porque la tabla de la política sigue diciendo que `/web` es `default_t`. Un reetiquetado nocturno, una actualización de política o un compañero aplicado rompen la web. `chcon` no es una solución, es un parche.

4. La solución: regla + aplicar.

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
sudo semanage fcontext -l | grep '^/web'
sudo restorecon -Rv /web
matchpathcon /web/index.html
curl http://localhost:82
```

Salida esperada:
```text
/web(/.*)?                                         all files          system_u:object_r:httpd_sys_content_t:s0
Relabeled /web from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /web/index.html from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
/web/index.html	system_u:object_r:httpd_sys_content_t:s0
<h1>Portal institucional - servido desde /web</h1>
```
Qué observar: `matchpathcon` muestra lo que la política **dice** que debe tener una ruta; `ls -Z` muestra lo que **tiene**. Cuando difieren, `restorecon`. La expresión `(/.*)?` cubre el directorio y todo lo que contenga. (`matchpathcon` está marcado como obsoleto en libselinux y puede imprimir un aviso de deprecación; la forma moderna de la misma consulta es `sudo restorecon -n -v /web/index.html`, que dice qué haría **sin** cambiar nada.)

5. Menciones (no ejecutar): `sudo fixfiles -F onboot` equivale a `touch /.autorelabel` y reetiqueta el sistema completo en el próximo arranque (minutos). `semanage fcontext -d "/web(/.*)?"` borra la regla. `semanage fcontext -l -C` lista solo las reglas propias.

- **Checkpoint:** pegar en el chat la salida de
```bash
ls -Zd /web; sudo semanage fcontext -l -C
```

### Lab 3.3 — Booleanos (5 min)

- **Objetivo:** activar una funcionalidad opcional de la política de forma persistente.

1. Explorar.

```bash
getsebool -a | grep -c '^httpd'
getsebool httpd_can_network_connect httpd_enable_homedirs ftpd_anon_write
sudo semanage boolean -l | grep -E '^(httpd_can_network_connect|httpd_enable_homedirs|ftpd_anon_write) '
```

Salida esperada:
```text
4x
httpd_can_network_connect --> off
httpd_enable_homedirs --> off
ftpd_anon_write --> off
ftpd_anon_write                (off  ,  off)  Allow ftpd to anon write
httpd_can_network_connect      (off  ,  off)  Allow httpd to can network connect
httpd_enable_homedirs          (off  ,  off)  Allow httpd to enable homedirs
```
Qué observar: `(actual, persistente)`. Apache tiene unos 40 interruptores. `httpd_can_network_connect` es el caso real más frecuente: Apache como proxy inverso hacia una aplicación en otro puerto/servidor devuelve 503 hasta activarlo.

2. Activar de forma persistente y volver al valor por defecto.

```bash
sudo setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect
sudo semanage boolean -l -C
sudo setsebool -P httpd_can_network_connect off
```

Salida esperada:
```text
httpd_can_network_connect --> on
SELinux boolean                State  Default Description
httpd_can_network_connect      (on   ,   on)  Allow httpd to can network connect
```
Qué observar: `-P` tarda unos segundos porque recompila la política; sin `-P` el cambio es inmediato pero se pierde al reiniciar. Lo dejamos en `off`: hoy no hay proxy inverso, y activar booleanos "por si acaso" amplía la superficie.

- **Checkpoint:** `getsebool httpd_can_network_connect` → `off`.

### Lab 3.4 — Provocar un AVC, leerlo con sealert y permissive por dominio (15 min)

- **Objetivo:** seguir el flujo completo de diagnóstico: síntoma → AVC → traducción → decisión → corrección → verificación.

1. Provocar una denegación realista: un archivo de configuración preparado en `/root` y movido al sitio.

```bash
sudo -i
echo "parametros internos del portal" > /root/config.txt
mv /root/config.txt /web/
exit
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82/config.txt
```

Salida esperada: `403`.

2. Leer el AVC crudo.

```bash
sudo ausearch -m AVC -ts recent
```

Salida esperada (resumida):
```text
----
time->Wed Sep  3 11:05:12 2026
type=PROCTITLE msg=audit(...): proctitle=2F7573722F7362696E2F687474706400...
type=SYSCALL msg=audit(...): arch=c000003e syscall=262 success=no exit=-13 ... comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0
type=AVC msg=audit(...): avc:  denied  { getattr } for  pid=2530 comm="httpd" path="/web/config.txt" dev="dm-0" ino=... scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file permissive=0
```
Leer en voz alta con la tabla: `comm=httpd` intentó `getattr` sobre un `file` cuyo `tcontext` es `admin_home_t`. Un archivo de `/root` dentro de `/web`. Decisión: contexto mal → `restorecon`. (En aarch64 el `arch=` y el número de `syscall` son otros; no importan para el diagnóstico.)

3. La versión traducida.

```bash
sudo journalctl -t setroubleshoot --since "5 min ago" --no-pager
```

Salida esperada:
```text
sep 03 11:05:13 rhel01 setroubleshoot[2610]: SELinux is preventing /usr/sbin/httpd from getattr access on the file /web/config.txt. For complete SELinux messages run: sealert -l 4c1f7a2e-....
```

Copiar el UUID y pedir el informe:

```bash
sudo sealert -l 4c1f7a2e-....   # pegar el UUID real
```

Salida esperada (resumida):
```text
SELinux is preventing /usr/sbin/httpd from getattr access on the file /web/config.txt.

*****  Plugin restorecon (99.5 confidence) suggests   ************************

If you want to fix the label.
/web/config.txt default label should be httpd_sys_content_t.
Then you can run restorecon.
Do
# /sbin/restorecon -v /web/config.txt

*****  Plugin catchall (1.49 confidence) suggests   **************************

If you believe that httpd should be allowed getattr access on the config.txt file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'httpd' --raw | audit2allow -M my-httpd
# semodule -X 300 -i my-httpd.pp
...
```
Qué observar: sealert ordena las sugerencias por confianza. La de 99.5 % es la correcta. La de "catchall" con `audit2allow` **siempre aparece** y **casi nunca** es la correcta: generaría una regla que permite a Apache leer archivos de `/root` para siempre. Si `journalctl -t setroubleshoot` no muestra nada (el daemon tarda o auditd no cargó el plugin), la alternativa que siempre funciona es:

```bash
sudo sealert -a /var/log/audit/audit.log | less
```

4. Corregir y verificar.

```bash
sudo restorecon -v /web/config.txt
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:82/config.txt
```

Salida esperada:
```text
Relabeled /web/config.txt from unconfined_u:object_r:admin_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
200
```

5. Permissive **por dominio**: la alternativa profesional a `setenforce 0` cuando una aplicación nueva da problemas y hay que seguir operando mientras se analiza.

```bash
sudo semanage permissive -a httpd_t
sudo semanage permissive -l
getenforce
sudo semanage permissive -d httpd_t
```

Salida esperada (resumida; `semanage permissive -a` tarda unos segundos porque compila un módulo de política):
```text
Customized Permissive Types

httpd_t

Builtin Permissive Types
...
Enforcing
```
Qué observar: el sistema sigue en Enforcing; solo `httpd_t` deja de ser bloqueado (y sigue registrando AVC). El resto de servicios quedan protegidos. Se quita con `-d` cuando se resolvió el problema.

6. Si sobra tiempo: ver qué **generaría** `audit2allow` sin instalar nada.

```bash
sudo ausearch -m AVC -ts recent | audit2allow
```

Salida esperada (aproximada):
```text
#============= httpd_t ==============
allow httpd_t admin_home_t:file getattr;
allow httpd_t default_t:file getattr;
allow httpd_t reserved_port_t:tcp_socket name_bind;
```
Qué observar: tres reglas que "arreglan" tres síntomas y abren tres huecos. `audit2allow -M` y `semodule -i` solo tienen sentido para software propio, tras revisar cada línea, y nunca para reemplazar un `restorecon` o un `semanage port`.

- **Checkpoint:** pegar en el chat la salida de
```bash
ls -Z /web/config.txt; getenforce; sudo semanage permissive -l | grep -c httpd_t
```
Debe mostrar `httpd_sys_content_t`, `Enforcing` y `0`.

---

## Bloque 4 — Hardening básico

### Conceptos (5 min)

Hardening es reducir lo que un atacante puede intentar y aumentar lo que veremos si lo intenta. Cuatro frentes, en orden de retorno por minuto invertido:

1. **Autenticación remota (SSH).** El 90 % de los intentos de intrusión contra un servidor Linux expuesto son fuerza bruta contra SSH. Llaves en vez de contraseñas, root fuera, lista de usuarios permitidos, pocos intentos. En RHEL 9 `sshd_config` incluye `/etc/ssh/sshd_config.d/*.conf` **al principio**, y en sshd **la primera aparición de una directiva gana**: por eso nuestros ajustes van en un archivo de ese directorio, y hay que vigilar qué otros archivos hay ahí.
2. **Contraseñas y bloqueo.** Complejidad con `pwquality`, bloqueo por intentos fallidos con `faillock` (activado vía `authselect`, la herramienta de RHEL 9 para configurar PAM sin editarlo a mano), caducidad con `chage`.
3. **Superficie.** Servicios que corren sin que nadie los use, puertos abiertos "porque venían así" (cockpit), paquetes sin actualizar. `dnf-automatic` para parches de seguridad sin intervención.
4. **Visibilidad.** auditd ya está registrando (lo usamos todo el día con `ausearch`); sudo registra cada comando; un banner legal; OpenSCAP para medir contra un estándar (CIS) en lugar de opinar.

Y lo que **no** haremos: deshabilitar SELinux ni firewalld "para que funcione", que es exactamente lo que el reto de hoy prohíbe.

### Lab 4.1 — sshd endurecido y probado desde el host (12 min)

- **Objetivo:** aplicar una configuración de sshd segura sin quedarse fuera, verificándola desde el equipo propio.

**Secuencia segura (leerla completa antes de teclear):** (a) confirmar que la llave funciona; (b) **no cerrar** la sesión actual; (c) aplicar y recargar; (d) probar en una **sesión nueva**; (e) solo entonces cerrar la vieja. Si algo sale mal, la consola de VirtualBox/UTM siempre está disponible para borrar el archivo y `systemctl reload sshd`.

1. (a) Desde el **host**, confirmar que la llave entra sin contraseña:

```bash
ssh -p 2222 -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no student@localhost hostname
```

Salida esperada: `rhel01`. Si pide contraseña o falla: `ssh-copy-id -p 2222 student@localhost` primero y repetir. **No seguir hasta que esto funcione.**

2. En la VM, ver qué hay en el directorio de fragmentos y confirmar el `Include`.

```bash
grep -n '^Include' /etc/ssh/sshd_config
ls -l /etc/ssh/sshd_config.d/
```

Salida esperada:
```text
19:Include /etc/ssh/sshd_config.d/*.conf
-rw-------. 1 root root 719 ... 50-redhat.conf
```
Qué observar: si además existe `01-permitrootlogin.conf` (lo crea el instalador cuando se marcó "permitir root por SSH con contraseña"), **ganaría** a cualquier `PermitRootLogin` posterior por orden alfabético. En ese caso: `sudo rm /etc/ssh/sshd_config.d/01-permitrootlogin.conf`.

3. Banner legal y archivo de hardening.

```bash
sudo tee /etc/issue.net > /dev/null <<'EOF'
******************************************************************
  Sistema institucional. Acceso restringido a personal autorizado.
  Toda actividad es registrada y auditada.
******************************************************************
EOF

sudo tee /etc/ssh/sshd_config.d/50-hardening.conf > /dev/null <<'EOF'
# Hardening basico de SSH - Dia 08
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers student
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
Banner /etc/issue.net
EOF
sudo chmod 600 /etc/ssh/sshd_config.d/50-hardening.conf
sudo sshd -t && echo "sintaxis OK"
sudo sshd -T | grep -iE '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|maxauthtries|allowusers|banner) '
```

Salida esperada:
```text
sintaxis OK
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
maxauthtries 3
allowusers student
banner /etc/issue.net
```
Qué observar: `sshd -T` muestra la configuración **efectiva** después de resolver todos los includes: es la prueba de que nuestro archivo ganó. `KbdInteractiveAuthentication no` cierra la puerta trasera de "contraseña vía PAM interactivo" que a veces queda abierta al poner solo `PasswordAuthentication no`. El nombre `50-hardening.conf` ordena antes que `50-redhat.conf` (h < r).

4. (c) Aplicar **sin cerrar esta sesión**.

```bash
sudo systemctl reload sshd
systemctl is-active sshd
```

Salida esperada: `active`.

5. (d) Desde el host, en una **terminal nueva**, tres pruebas:

```bash
ssh -p 2222 student@localhost hostname
ssh -p 2222 root@localhost
ssh -p 2222 -o PubkeyAuthentication=no student@localhost
```

Salida esperada:
```text
******************************************************************
  Sistema institucional. Acceso restringido a personal autorizado.
  ...
rhel01
root@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
student@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```
Qué observar: el banner aparece **antes** de autenticar; root ya no puede; sin llave no hay forma de entrar (no ofrece `password`). En la VM, `sudo journalctl -u sshd --since "2 min ago"` muestra los rechazos. (e) Ahora sí se puede cerrar la sesión vieja.

6. **Si sobra tiempo — SSH en un segundo puerto con SELinux.** (⚠️ Verificar en la VM antes de la clase que `sudo semanage port -l | grep -w 2222` no devuelve nada; si el puerto ya estuviera etiquetado, usar otro, por ejemplo 2022.) Agregar `Port 22` y `Port 2222` a `50-hardening.conf` (sshd escucha en ambos, así el 22 sigue como respaldo), `sudo systemctl restart sshd`, y ver en `sudo ausearch -m AVC -ts recent | grep sshd` la denegación `name_bind` sobre el 2222. Corregir con `sudo semanage port -a -t ssh_port_t -p tcp 2222`, reiniciar sshd, `sudo ss -tlnp | grep sshd` muestra ambos puertos, y abrir con `sudo firewall-cmd --permanent --zone=internal --add-port=2222/tcp && sudo firewall-cmd --reload`. Probar desde el host: `ssh -p 2222 student@192.168.56.10`. Quitar la línea `Port 2222` al terminar para no confundir con el port forwarding.

- **Checkpoint:** pegar en el chat la salida de
```bash
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|maxauthtries) '
```

### Lab 4.2 — Contraseñas: pwquality, faillock y chage (12 min)

- **Objetivo:** exigir contraseñas fuertes, bloquear tras 5 intentos fallidos y desbloquear, y fijar caducidad.

> **Nota de ritmo (12 min son justos):** los pasos 1–3 (pwquality) y 4–6 (faillock) son los que hay que teclear. El paso 7 (`chage`) se demuestra en pantalla y queda como tarea si el reloj aprieta; se practicó en el Día 03. El instructor pide que **todos escriban las contraseñas de prueba en el chat antes de teclearlas**, para que nadie se invente una que sí pase y se pierda el efecto.

1. Usuario de pruebas (antes de endurecer la política, para que root le ponga una contraseña sin drama).

```bash
sudo useradd prueba
echo 'Prueba.2026!' | sudo passwd --stdin prueba
```

Salida esperada: `Changing password for user prueba.` / `passwd: all authentication tokens updated successfully.`

Qué observar: `passwd --stdin` es una extensión de Red Hat (cómoda, pero deja la contraseña en el historial del shell). La forma portable y equivalente es `echo 'prueba:Prueba.2026!' | sudo chpasswd`. En un servidor real, ninguna de las dos: `sudo passwd prueba` interactivo.

2. Política de complejidad.

```bash
sudo cp /etc/security/pwquality.conf /etc/security/pwquality.conf.bak
sudo tee -a /etc/security/pwquality.conf > /dev/null <<'EOF'

# Politica institucional - Dia 08
minlen = 12
minclass = 3
maxrepeat = 3
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
dictcheck = 1
EOF
grep -vE '^#|^$' /etc/security/pwquality.conf
```

Salida esperada: las **ocho** líneas de configuración agregadas (el archivo original de RHEL 9 viene con todo comentado, así que `grep -vE '^#|^$'` solo muestra lo nuestro). Qué observar: `minlen` mínimo 12; `dcredit=-1` significa "al menos un dígito" (negativo = requisito; positivo = bonificación); `minclass=3` exige tres clases de caracteres — con los cuatro `credit` en `-1` ya se exigen las cuatro clases, así que `minclass` queda redundante: se deja porque es lo que piden los estándares y porque, si mañana se quitan los `credit`, el mínimo de clases sigue en pie. Aplica de inmediato: pwquality se lee en cada `passwd`. Para root solo avisa (root puede insistir y la contraseña se acepta); para los usuarios normales bloquea.

3. Probarla como el usuario (contraseña actual `Prueba.2026!`; como nueva contraseña intentar `hola123`, después `PanamaTech2026` y después cualquier otra corta). pwquality comprueba primero la longitud: por eso la segunda prueba tiene 14 caracteres, para que el error sea el del carácter especial.

```bash
su - prueba
passwd
exit
```

Salida esperada:
```text
Changing password for user prueba.
Current password:
New password:
BAD PASSWORD: The password is shorter than 12 characters
New password:
BAD PASSWORD: The password contains less than 1 non-alphanumeric characters
New password:
passwd: Have exhausted maximum number of retries for service
```

4. Activar faillock con authselect y ajustar umbrales.

```bash
sudo authselect current
sudo authselect enable-feature with-faillock
sudo authselect current
sudo sed -i 's/^# deny = 3/deny = 5/; s/^# unlock_time = 600/unlock_time = 900/' /etc/security/faillock.conf
grep -E '^(deny|unlock_time)' /etc/security/faillock.conf
```

Salida esperada (aproximada; ⚠️ Verificar en la VM antes de la clase: el perfil y la lista de features dependen de la versión exacta de RHEL 9):
```text
Profile ID: sssd
Enabled features: None

Profile ID: sssd
Enabled features:
- with-faillock

deny = 5
unlock_time = 900
```
Qué observar: el perfil puede llamarse distinto según la versión (`sssd` en la mayoría de RHEL 9; en 9.x recientes con authselect 1.5 aparece también `local`), y `Enabled features` puede venir con algo ya activado (`with-silent-lastlog`, `with-fingerprint`) o con `None`; lo importante es que después del comando aparezca `with-faillock`. Si `authselect current` responde "No existing configuration detected", ejecutar primero `sudo authselect select sssd --force` y repetir el `enable-feature`. En RHEL 9 los umbrales viven en `/etc/security/faillock.conf`, no en los archivos de PAM (que authselect genera y no se editan a mano). Si authselect se queja de "unexpected changes", ver la tabla de errores.

Comprobar que PAM quedó realmente con faillock:

```bash
grep -n faillock /etc/pam.d/system-auth
```

Salida esperada (resumida):
```text
6:auth        required                                     pam_faillock.so preauth silent
10:auth        required                                     pam_faillock.so authfail
13:account     required                                     pam_faillock.so
```

5. Cinco intentos fallidos (escribir una contraseña incorrecta las cinco veces) y observar el bloqueo.

```bash
for i in 1 2 3 4 5; do su - prueba -c true; done
sudo faillock --user prueba
```

Salida esperada:
```text
Password:
su: Authentication failure
(x5)
prueba:
When                Type  Source                                           Valid
2026-09-03 11:32:10 TTY   pts/0                                                V
2026-09-03 11:32:12 TTY   pts/0                                                V
2026-09-03 11:32:14 TTY   pts/0                                                V
2026-09-03 11:32:16 TTY   pts/0                                                V
2026-09-03 11:32:18 TTY   pts/0                                                V
```

Ahora **con la contraseña correcta**:

```bash
su - prueba -c 'echo ENTRE'
sudo journalctl --since "3 min ago" | grep -i 'faillock' | tail -2
```

Salida esperada: `su: Authentication failure` (bloqueado aunque la contraseña sea correcta) y en el journal `pam_faillock(su-l:auth): Consecutive login failures for user prueba account temporarily locked`.

6. Desbloquear y comprobar.

```bash
sudo faillock --user prueba --reset
sudo faillock --user prueba
su - prueba -c 'echo ENTRE'
```

Salida esperada: la lista vacía (solo el encabezado `prueba:`) y, tras la contraseña correcta, `ENTRE`. Qué observar: por defecto faillock no bloquea a root (`even_deny_root` comentado): bloquear a root remoto es deseable, pero solo si hay consola.

7. Caducidad con `chage`.

```bash
sudo chage -l prueba
sudo chage -M 90 -m 1 -W 14 prueba
sudo chage -l prueba | grep -E 'Maximum|Minimum|warning'
grep -E '^PASS_(MAX|MIN|WARN)' /etc/login.defs
```

Salida esperada (`chage -l` imprime primero el mínimo y después el máximo):
```text
Minimum number of days between password change          : 1
Maximum number of days between password change          : 90
Number of days of warning before password expires       : 14
PASS_MAX_DAYS   99999
PASS_MIN_DAYS   0
PASS_WARN_AGE   7
```
Qué observar: `/etc/login.defs` fija los valores para usuarios **nuevos**; `chage` para los existentes. `chage -d 0 usuario` obliga a cambiar la contraseña en el próximo inicio de sesión; `chage -E 2026-12-31 usuario` fija fecha de expiración de la cuenta (contratistas).

- **Checkpoint:** pegar en el chat la salida de
```bash
sudo authselect current | grep faillock; grep -E '^(deny|unlock_time)' /etc/security/faillock.conf
```

### Lab 4.3 — Superficie, actualizaciones y visibilidad (6 min)

- **Objetivo:** auditar qué corre y qué escucha, cerrar lo que no se usa, y dejar las actualizaciones de seguridad en automático.

> **Nota de ritmo (6 min):** el `dnf install -y dnf-automatic` del paso 3 debe estar **ya descargado** (`--downloadonly` en la preparación del Día 07 o en la apertura); si no, este lab no cabe en 6 minutos y el paso 3 pasa a demo del instructor. El paso 4 es lectura comentada, no hace falta que todos peguen la salida.

1. Auditoría de servicios y puertos.

```bash
systemctl list-units --type=service --state=running --no-pager
sudo ss -tulpn
```

Salida esperada (resumida):
```text
  UNIT                     LOAD   ACTIVE SUB     DESCRIPTION
  auditd.service           loaded active running Security Auditing Service
  chronyd.service          loaded active running NTP client/server
  crond.service            loaded active running Command Scheduler
  firewalld.service        loaded active running firewalld - dynamic firewall daemon
  httpd.service            loaded active running The Apache HTTP Server
  NetworkManager.service   loaded active running Network Manager
  rsyslog.service          loaded active running System Logging Service
  sshd.service             loaded active running OpenSSH server daemon
  ...
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0          127.0.0.1:323        0.0.0.0:*     users:(("chronyd",pid=780,fd=5))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=900,fd=3))
tcp   LISTEN 0      511                *:82               *:*     users:(("httpd",pid=2600,fd=4),...)
```
Qué observar: `ss -tulpn` es la vista del atacante desde adentro: cada `LISTEN` en `0.0.0.0` o `*` es un servicio expuesto. Un servicio que solo se usa localmente debería escuchar en `127.0.0.1` (como chronyd). Lo que no se usa: `sudo systemctl disable --now servicio` (y `mask` si no debe arrancar nunca).

2. Cerrar en el firewall lo que nadie pidió: cockpit (9090) está abierto por defecto aunque no esté instalado.

```bash
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --permanent --zone=internal --remove-service=cockpit
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

Salida esperada: `success` x3 y `dhcpv6-client http ssh`.

3. Actualizaciones de seguridad automáticas.

```bash
sudo dnf install -y dnf-automatic        # ya instalado en el Lab 1.1: dirá "Nothing to do"
sudo cp /etc/dnf/automatic.conf /etc/dnf/automatic.conf.bak
sudo sed -i 's/^apply_updates = no/apply_updates = yes/; s/^upgrade_type = default/upgrade_type = security/' /etc/dnf/automatic.conf
grep -E '^(apply_updates|upgrade_type|download_updates)' /etc/dnf/automatic.conf
sudo systemctl enable --now dnf-automatic.timer
systemctl list-timers 'dnf-automatic*' --no-pager
```

Salida esperada:
```text
upgrade_type = security
download_updates = yes
apply_updates = yes
NEXT                        LEFT     LAST PASSED UNIT                ACTIVATES
Thu 2026-09-04 06:41:03 EST 19h left -    -      dnf-automatic.timer dnf-automatic.service
```
Qué observar: `upgrade_type = security` aplica solo erratas de seguridad (menos riesgo de cambios funcionales). No reinicia el servidor: `sudo dnf needs-restarting -r` dice si hace falta (si responde `No such command`, falta el paquete: `sudo dnf install -y dnf-plugins-core`). Para saber qué hay pendiente hoy: `sudo dnf updateinfo list security`. El paquete `dnf-automatic` trae varios timers (`dnf-automatic.timer`, `dnf-automatic-install.timer`, `dnf-automatic-download.timer`, `dnf-automatic-notifyonly.timer`); se habilita **uno solo** —el nuestro es `dnf-automatic.timer`, que obedece a lo que diga `automatic.conf`.

4. Visibilidad: lo que ya se registra sin que hagamos nada.

```bash
sudo aureport -au --failed | tail -5
sudo grep COMMAND /var/log/secure | tail -3
umask
sudo bash -c 'umask'
grep -E '^UMASK' /etc/login.defs
```

Salida esperada (resumida):
```text
1. 09/03/2026 11:32:10 prueba ? /dev/pts/0 /usr/bin/su no 1520
...
Sep  3 11:40:02 rhel01 sudo[3105]:  student : TTY=pts/0 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/firewall-cmd --reload
0002
0022
UMASK			022
```
Qué observar: auditd guardó cada intento fallido de `su` (sirve para reconstruir un incidente); sudo deja quién, dónde y qué comando en `/var/log/secure` (`journalctl _COMM=sudo` también).

Sobre el `umask`: en RHEL, `/etc/profile` y `/etc/bashrc` aplican **002** a los usuarios con UID ≥ 200 cuyo grupo primario se llama igual que el usuario (el esquema *User Private Group*, que es el caso de `student`) y **022** a todos los demás, incluido root. Por eso `umask` como `student` responde `0002` y como root `0022`; `/etc/login.defs` declara `UMASK 022`, que es lo que usan `useradd` y los servicios que no pasan por el shell interactivo. Los perfiles CIS piden **027** y para eso hay que cambiarlo en los tres sitios (`/etc/login.defs`, `/etc/profile`, `/etc/bashrc`). ⚠️ Verificar en la VM antes de la clase: ejecutar los tres comandos y ajustar los valores del ejemplo a lo que devuelva la instalación real.

Menciones para la libreta (no se ejecutan hoy): reglas persistentes de auditd en `/etc/audit/rules.d/*.rules` (`-w /etc/passwd -p wa -k identity`); `Defaults log_output` en `/etc/sudoers.d/` para grabar sesiones (`sudoreplay`); `fail2ban` desde EPEL para bloquear IPs por fuerza bruta (faillock protege la cuenta, fail2ban la IP; en RHEL 9 EPEL requiere habilitar el repositorio CRB); `fapolicyd` para permitir ejecutar solo binarios de confianza (potente y delicado: puede bloquear scripts propios).

- **Checkpoint:** pegar en el chat la salida de
```bash
systemctl is-active dnf-automatic.timer; sudo firewall-cmd --list-services
```

### Lab 4.4 — OpenSCAP: medir en vez de opinar (demo del instructor, 5 min)

- **Objetivo:** ver cómo se evalúa el servidor contra el benchmark CIS y cómo se lee el reporte. El instructor ejecuta el escaneo antes de clase (tarda varios minutos) y muestra el reporte ya generado; los participantes **no** lo ejecutan en clase (lo repiten como tarea). Los pasos 1 y 2 quedan aquí como referencia para esa tarea.

1. Instalar el escáner y las guías.

```bash
sudo dnf install -y openscap-scanner scap-security-guide
ls /usr/share/xml/scap/ssg/content/
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml | grep -A1 'Title: CIS'
```

Salida esperada (resumida):
```text
ssg-rhel9-ds.xml  ssg-rhel9-ocil.xml  ssg-rhel9-xccdf.xml ...
			Title: CIS Red Hat Enterprise Linux 9 Benchmark for Level 2 - Server
				Id: xccdf_org.ssgproject.content_profile_cis
			Title: CIS Red Hat Enterprise Linux 9 Benchmark for Level 1 - Server
				Id: xccdf_org.ssgproject.content_profile_cis_server_l1
			...
```

2. Evaluar con el perfil CIS Server Level 1 y generar reporte HTML (3–6 minutos).

```bash
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /tmp/resultados.xml \
  --report /tmp/reporte.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
grep -c '<result>fail</result>' /tmp/resultados.xml
```

Salida esperada (resumida):
```text
Title   Ensure /tmp Located On Separate Partition
Rule    xccdf_org.ssgproject.content_rule_partition_for_tmp
Ident   CCE-83474-7
Result  fail

Title   Disable SSH Root Login
Rule    xccdf_org.ssgproject.content_rule_sshd_disable_root_login
Result  pass
...
1xx
```
Qué observar: el código de salida es 2 cuando hay reglas en `fail`; no es un error. Varias reglas que acabamos de aplicar (root SSH, faillock, banner) salen en `pass`. Muchas de las que fallan son de particionado o de kernel y **no** se corrigen a ciegas.

3. Ver el reporte desde el host.

```bash
scp -P 2222 student@localhost:/tmp/reporte.html ~/Desktop/reporte-rhel01.html
```

Abrir el HTML en el navegador: puntuación, reglas por severidad, y para cada regla la descripción, la justificación y el comando de corrección propuesto. Mención: `oscap xccdf generate fix --profile ... --fix-type bash --output /tmp/remediacion.sh ssg-rhel9-ds.xml` genera un script de remediación que **se revisa antes de ejecutar**; `--remediate` lo aplica en vivo y en un servidor real puede dejar sin acceso (cambia sshd, particiones, kernel). En la institución, el reporte es el insumo para un plan, no un botón.

- **Checkpoint (solo instructor):** `ls -lh /tmp/reporte.html` y la pantalla del reporte compartida.

### Checklist de buenas prácticas (15 puntos, para imprimir)

| # | Práctica | Cómo se verifica hoy |
|---|---|---|
| 1 | SELinux en `Enforcing`, siempre | `getenforce` |
| 2 | firewalld activo; solo servicios/puertos necesarios en cada zona | `firewall-cmd --list-all-zones` |
| 3 | Red de administración en zona propia (`internal`) con origen explícito | `firewall-cmd --get-active-zones` |
| 4 | Cockpit y servicios no usados fuera del firewall y deshabilitados | `ss -tulpn`, `systemctl list-units --state=running` |
| 5 | SSH sin root, sin contraseñas, con `AllowUsers` y `MaxAuthTries` bajo | `sshd -T` |
| 6 | Llaves SSH para administradores; contraseñas solo para consola | `~/.ssh/authorized_keys` |
| 7 | Banner legal antes del login | `ssh` muestra `/etc/issue.net` |
| 8 | Complejidad de contraseñas (`minlen ≥ 12`, 3 clases) | `/etc/security/pwquality.conf` |
| 9 | Bloqueo por intentos fallidos (`faillock`, `deny = 5`) | `authselect current`, `faillock --user` |
| 10 | Caducidad de contraseñas y cuentas de terceros con fecha de fin | `chage -l` |
| 11 | Actualizaciones de seguridad automáticas y reinicio planificado | `systemctl list-timers dnf-automatic*`, `dnf needs-restarting -r` |
| 12 | auditd activo; `ausearch` como primera herramienta ante un incidente | `systemctl is-active auditd` |
| 13 | Nadie trabaja como root; todo pasa por `sudo` y queda en `/var/log/secure` | `grep COMMAND /var/log/secure` |
| 14 | Contextos SELinux persistentes con `semanage fcontext`, nunca solo `chcon` | `semanage fcontext -l -C` |
| 15 | Línea base medida con OpenSCAP (CIS L1) y plan de remediación revisado | `/tmp/reporte.html` |

---

## Reto individual (20 min)

Antes de empezar, cada participante toma un snapshot `dia08-pre-reto` y luego ejecuta el script que el instructor comparte por el chat (`sudo bash romper-dia08.sh`). Después recibe este ticket:

```text
Ticket PGN-2026-0812 — Prioridad ALTA
"La web institucional dejó de cargar desde las computadoras de la oficina.
El proveedor dice que ahora publica en el puerto 8082 y que el contenido está
en /sitio. Desde el propio servidor a veces da 403 y a veces ni responde.
Necesitamos que cargue desde afuera en http://192.168.56.10:8082 hoy."

Condiciones:
- Prohibido: setenforce 0, SELINUX=permissive/disabled, systemctl stop firewalld,
  zona trusted, chcon como solución final.
- Entregable: la URL cargando desde el navegador del equipo propio, y un texto
  con cuatro líneas por cada causa encontrada: qué estaba mal, cómo lo
  encontré, cómo lo corregí, cómo lo verifiqué.
```

### Script del instructor: `romper-dia08.sh`

```bash
#!/bin/bash
# romper-dia08.sh — Ticket del Día 08. Ejecutar con sudo en la VM del participante.
# No usa set -e a propósito: algunos pasos pueden fallar sin importancia.

CONF=/etc/httpd/conf/httpd.conf

# 1) Apache al puerto 8082 SIN registrarlo en SELinux
sed -i 's/^Listen .*/Listen 8082/' "$CONF"

# 2) DocumentRoot en /sitio con contexto default_t (sin regla fcontext)
mkdir -p /sitio
echo "<h1>Portal institucional - rhel01 - puerto 8082</h1>" > /sitio/index.html
chmod 755 /sitio; chmod 644 /sitio/index.html
sed -i 's#^DocumentRoot .*#DocumentRoot "/sitio"#' "$CONF"
sed -i -E 's#^<Directory "(/var/www/html|/web)">#<Directory "/sitio">#' "$CONF"
restorecon -R /sitio

# 3) Firewall: cerrar lo abierto hoy y mandar la red host-only a la zona drop
firewall-cmd --permanent --remove-service=http                         >/dev/null 2>&1
firewall-cmd --permanent --remove-port=82/tcp                          >/dev/null 2>&1
firewall-cmd --permanent --zone=internal --remove-service=http         >/dev/null 2>&1
firewall-cmd --permanent --zone=internal --remove-port=82/tcp          >/dev/null 2>&1
firewall-cmd --permanent --zone=internal --remove-source=192.168.56.0/24 >/dev/null 2>&1
firewall-cmd --permanent --zone=drop --add-source=192.168.56.0/24      >/dev/null
firewall-cmd --reload                                                  >/dev/null

# 4) Reiniciar Apache (fallará: es parte del ticket)
systemctl restart httpd 2>/dev/null
echo "Ticket entregado. Suerte."
```

### Solución (para el instructor)

**Capa 1 — el servicio no arranca (SELinux, puerto).**

```bash
systemctl status httpd --no-pager -l | grep -E 'Active|Permission'
sudo ausearch -m AVC -ts recent | grep name_bind | tail -1
sudo semanage port -a -t http_port_t -p tcp 8082
# Si responde "ValueError: Port tcp/8082 already defined" (es us_cli_port_t):
sudo semanage port -m -t http_port_t -p tcp 8082
sudo systemctl restart httpd; systemctl is-active httpd
```

Verificación: `active`; `sudo semanage port -l | grep -w http_port_t` incluye 8082.

**Capa 2 — 403 desde adentro (SELinux, contexto).**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8082      # 403
ls -Zd /sitio                                                        # default_t
sudo ausearch -m AVC -ts recent | grep sitio | tail -1               # tcontext=default_t
sudo semanage fcontext -a -t httpd_sys_content_t "/sitio(/.*)?"
sudo restorecon -Rv /sitio
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8082      # 200
```

**Capa 3 — no carga desde afuera (firewalld, zona y puerto).**

```bash
sudo firewall-cmd --get-active-zones           # drop con sources: 192.168.56.0/24
sudo firewall-cmd --permanent --zone=drop --remove-source=192.168.56.0/24
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24
sudo firewall-cmd --permanent --zone=internal --add-port=8082/tcp
sudo firewall-cmd --permanent --zone=public --add-port=8082/tcp      # para la ruta NAT, si se agregó forwarding 8082
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
```

Verificación: desde el host, `http://192.168.56.10:8082` carga. Quien no tenga host-only: agregar port forwarding 8082→8082 en VirtualBox y `http://localhost:8082`.

**Errores que se verán:** poner `setenforce 0` "para probar" y olvidarse (descalifica); abrir 8082 solo en `public` y no entender por qué la ruta host-only sigue fallando (el origen está en `drop`: `--get-active-zones` es el primer comando); usar `chcon` y dejarlo así; olvidar `--reload`; buscar el problema en `/var/log/httpd/error_log` (útil, pero el 403 por SELinux se ve mejor en `ausearch`). Puntos extra a quien documente las tres capas en el orden del método del Día 10: servicio → logs → red/firewall → permisos → SELinux.

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `firewall-cmd --state` | Confirmar que firewalld corre |
| `firewall-cmd --get-default-zone` / `--get-active-zones` | Zona por defecto / zonas con interfaces u orígenes asignados |
| `firewall-cmd --list-all` / `--list-all-zones` | Reglas de la zona por defecto / de todas |
| `firewall-cmd --get-services` / `--info-service=http` | Servicios predefinidos y sus puertos |
| `firewall-cmd --permanent --add-service=http && firewall-cmd --reload` | Abrir un servicio de forma persistente |
| `firewall-cmd --permanent --add-port=8082/tcp` | Abrir un puerto sin servicio predefinido |
| `firewall-cmd --remove-service=X` / `--remove-port=X/tcp` | Cerrar (agregar `--permanent` y `--reload`) |
| `firewall-cmd --runtime-to-permanent` | Guardar lo probado en runtime |
| `firewall-cmd --query-service=http` | Sí/no con código de salida, para scripts |
| `firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24` | Clasificar una red de origen en una zona |
| `firewall-cmd --permanent --zone=Z --add-rich-rule='rule family="ipv4" source address="A" service name="ssh" log prefix="P " accept'` | Regla enriquecida con log |
| `firewall-cmd --permanent --zone=internal --change-interface=enp0s8` | Mover una interfaz de zona (sin `--permanent` es solo runtime; lo persistente de verdad es `nmcli connection modify <conexión> connection.zone internal`) |
| `nft list ruleset` | Ver las reglas nftables que firewalld generó |
| `getenforce` / `setenforce 0` \| `setenforce 1` / `sestatus` | Modo SELinux actual, cambio temporal, estado completo |
| `ls -Z`, `ls -Zd`, `ps -eZ`, `id -Z` | Contextos de archivos, directorios, procesos y sesión |
| `restorecon -Rv /ruta` | Reaplicar el contexto que la política dicta |
| `matchpathcon /ruta` | Qué contexto **debería** tener una ruta |
| `semanage port -l \| grep -w http_port_t` | Puertos etiquetados para un tipo |
| `semanage port -a -t http_port_t -p tcp 82` (`-m` si ya definido, `-d` para borrar) | Etiquetar un puerto para un servicio |
| `semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"` | Regla persistente de contexto para una ruta |
| `semanage fcontext -l -C` / `semanage port -l -C` | Solo personalizaciones locales |
| `getsebool -a \| grep httpd` / `semanage boolean -l` | Ver booleanos |
| `setsebool -P httpd_can_network_connect on` | Activar booleano de forma persistente |
| `ausearch -m AVC -ts recent` | Denegaciones recientes en crudo |
| `journalctl -t setroubleshoot` / `sealert -l UUID` / `sealert -a /var/log/audit/audit.log` | Denegaciones traducidas con sugerencia |
| `semanage permissive -a httpd_t` / `-d` | Permissive para un solo dominio |
| `sshd -t` / `sshd -T \| grep -i permitrootlogin` | Validar sintaxis / ver configuración efectiva de sshd |
| `authselect enable-feature with-faillock` | Activar bloqueo por intentos fallidos |
| `faillock --user X` / `faillock --user X --reset` | Ver intentos / desbloquear |
| `chage -l X` / `chage -M 90 -W 14 X` | Ver y fijar caducidad de contraseña |
| `ss -tulpn` / `systemctl list-units --type=service --state=running` | Auditar puertos y servicios |
| `systemctl enable --now dnf-automatic.timer` | Actualizaciones automáticas |
| `oscap xccdf eval --profile ..._cis_server_l1 --report /tmp/r.html ssg-rhel9-ds.xml` | Evaluar contra CIS |

---

## Notas para el instructor

- **Preparar antes de la clase:**
  - Correr el día completo en la VM propia la noche anterior partiendo de `dia07-fin`, en especial los Labs 1.2 (paso 4, el cable de vida por host-only), 3.1 y 4.1. Tomar nota de los nombres de interfaz de UTM y de la red host-only real (ver Diferencias).
  - Descargar los paquetes en la VM de clase (y pedir a los participantes que lo hagan durante la apertura si el Día 07 no lo dejó de tarea): `sudo dnf install -y --downloadonly httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic openscap-scanner scap-security-guide setools-console`. `setroubleshoot-server` arrastra ~100 MB de dependencias; por eso el Lab 1.1 lo instala junto con httpd (mientras se explica), y el Lab 3.1 solo lo confirma.
  - Verificar la afirmación clave del día sobre puertos: `sudo semanage port -l | grep -wE '82|8080|8082'` y, con `setools-console`, `sesearch -A -s httpd_t -t http_cache_port_t -c tcp_socket -p name_bind` (debe mostrar la regla `allow`: por eso 8080 no falla y usamos 82). ⚠️ Verificar en la VM antes de la clase: si en la versión instalada 8082 **no** aparece como `us_cli_port_t`, la solución del reto usa `-a` y no `-m`; ajustar el discurso.
  - ⚠️ Verificar en la VM antes de la clase, y anotar el valor real para no improvisar en vivo, estas cinco cosas que cambian entre versiones/instalaciones de RHEL 9:
    1. `authselect current` → nombre del perfil (`sssd` o `local`) y features ya activadas (Lab 4.2 paso 4).
    2. `umask` como `student`, `sudo bash -c 'umask'` y `grep ^UMASK /etc/login.defs` (Lab 4.3 paso 4).
    3. `sudo mkdir /pruebaselinux && ls -Zd /pruebaselinux` → confirmar que un directorio nuevo creado en `/` queda en `default_t` (Lab 3.2 paso 1); `sudo rmdir /pruebaselinux` después.
    4. `sudo nft list ruleset | grep dport` → formato real de las líneas (Lab 1.1 paso 8).
    5. `sudo semanage port -l | grep -w 2222` → que no esté ya etiquetado (Lab 4.1 paso 6).
  - Ejecutar el escaneo OpenSCAP (Lab 4.4) antes de clase y tener `reporte.html` abierto en el navegador del host.
  - Probar `romper-dia08.sh` sobre un snapshot al final del recorrido del día y confirmar que la solución completa deja `http://192.168.56.10:8082` cargando. Restaurar el snapshot.
  - Tener la ventana de consola de la VM visible (no solo SSH) durante los Labs 1.2 y 4.1, y recordarlo a los participantes: es la salida de emergencia si alguien se bloquea.
  - Agregar en VirtualBox los port forwardings de respaldo para quien no tenga host-only: 18082→82 y 8082→8082 (se pueden agregar con la VM encendida).
  - Confirmar en cada participante, en la apertura, que `ssh -p 2222 student@localhost` entra **sin contraseña**. Quien no lo tenga hace `ssh-copy-id` en ese momento; sin llave, el Lab 4.1 se hace solo hasta el paso 3 (sin aplicar).

- **Qué estudiar si es nuevo en RHEL:**
  1. `semanage` en sus cuatro sabores (`port`, `fcontext`, `boolean`, `permissive`): `man semanage-port`, `man semanage-fcontext`. Practicar el ciclo completo con el 82 y con `/web`, incluido el error "already defined" y el `-m`.
  2. `sealert`/setroubleshoot: instalar, reiniciar auditd con `service auditd restart`, provocar el 403 y ver cuánto tarda en aparecer en `journalctl -t setroubleshoot`. Practicar `sealert -a /var/log/audit/audit.log` como plan B. `man sealert`.
  3. Zonas de firewalld y la precedencia origen > interfaz: `man firewalld.zones`, `man firewalld.richlanguage`. Reproducir el paso 2 del Lab 1.2 (la ruta host-only deja de funcionar al asignar el origen a internal) hasta poder explicarlo sin mirar.
  4. `authselect` y `faillock`: `man authselect`, `man faillock.conf`, `man pam_faillock`. Ver qué escribe `authselect enable-feature with-faillock` en `/etc/pam.d/system-auth` (`grep faillock`).
  5. El orden de `Include` en sshd y "la primera directiva gana": `man sshd_config` (sección Include). Jugar con `sshd -T` cambiando el nombre del archivo a `60-hardening.conf` y ver qué cambia.
  6. OpenSCAP: correr una vez el escaneo y abrir el reporte; leer tres reglas en `fail` y su remediación propuesta. `man oscap`.

- **Errores frecuentes de los participantes y cómo resolverlos:**

| Síntoma | Causa | Solución |
|---|---|---|
| "Puse `--permanent --add-service=http` y no carga" | Falta `--reload` | `sudo firewall-cmd --reload`; enseñar `--list-services` vs `--permanent --list-services` |
| "Cargaba y después de reiniciar dejó de cargar" | Se abrió sin `--permanent` | `--runtime-to-permanent` o repetir con `--permanent` + `--reload` |
| "Desde `localhost:8080` carga pero desde `192.168.56.10` no" (o al revés) | El origen host-only cae en `internal`; el NAT en `public`; el servicio está en una sola zona | Abrir en ambas zonas o revisar `--get-active-zones` |
| `curl: (7) ... No route to host` desde el host | El firewall rechaza (zona sin el servicio) | `firewall-cmd --list-all` y `--zone=internal --list-all` |
| `curl: (7) ... Connection refused` | Apache no está escuchando en ese puerto (SELinux impidió el bind o Listen mal) | `systemctl status httpd`, `ss -tlnp` |
| `semanage: command not found` | Falta el paquete | `sudo dnf install policycoreutils-python-utils` |
| `ValueError: Port tcp/8080 already defined` | El puerto ya tiene otro tipo | `semanage port -m` en lugar de `-a` |
| "`restorecon` no cambia nada en `/web`" o "lo vuelve a `default_t`" | No hay regla fcontext; se usó `chcon` | `semanage fcontext -a ...` y luego `restorecon -Rv` |
| 403 persiste con contexto correcto | Falta `Require all granted` para el nuevo DocumentRoot | Revisar el bloque `<Directory>`; `apachectl configtest`; `/var/log/httpd/error_log` |
| `Permission denied (publickey)` tras el Lab 4.1 | La llave no estaba instalada o `AllowUsers` no incluye al usuario | Consola de la VM: `sudo rm /etc/ssh/sshd_config.d/50-hardening.conf && sudo systemctl reload sshd`; `ssh-copy-id`; repetir |
| Root sigue entrando por SSH | Existe `01-permitrootlogin.conf` (primera directiva gana) | `sudo rm /etc/ssh/sshd_config.d/01-permitrootlogin.conf`; `sshd -T \| grep permitrootlogin` |
| `authselect: [error] ... Unexpected changes to the PAM configuration` | Alguien editó PAM a mano | `sudo authselect select sssd with-faillock --force` |
| `journalctl -t setroubleshoot` vacío | El daemon aún no analizó, o auditd no cargó el plugin | Esperar 10 s; `sudo service auditd restart`; plan B `sealert -a /var/log/audit/audit.log` |
| "No sale ningún AVC en `ausearch`" | Denegación silenciada por reglas `dontaudit` | `sudo semodule -DB` (desactiva dontaudit), reproducir, `sudo semodule -B` para volver |
| `setsebool -P` "se queda colgado" | Está recompilando la política (5–20 s en la VM) | Esperar; no interrumpir |
| `systemctl restart auditd` → "Operation refused" | `RefuseManualStop=yes` en la unidad | `sudo service auditd restart` |
| `su - prueba` funciona a pesar de 5 fallos | faillock no activo (authselect sin `with-faillock`) o los fallos fueron contra otro usuario | `authselect current`; `faillock --user prueba` |
| El script del reto dejó sin SSH a alguien | Estaba conectado por la ruta host-only, ahora en `drop` (solo afecta conexiones nuevas) | Reconectar por NAT `ssh -p 2222 student@localhost` o usar la consola |

- **Diferencias VirtualBox (x86_64) vs UTM (aarch64):**
  - Interfaces: `enp0s3`/`enp0s8` en VirtualBox; en UTM típicamente `enp0s1`/`enp0s2`: siempre verificar con `nmcli device` antes de mostrar `--get-active-zones` o `--change-interface`.
  - Red host-only: en VirtualBox es 192.168.56.0/24 y la VM es `.10`. En UTM la red "Host Only" la asigna macOS (vmnet), con frecuencia en otro rango; usar `ip -4 addr show` en la VM y sustituir `192.168.56.0/24` en **todas** las reglas y en `romper-dia08.sh` por la red real. Los comandos no cambian, solo el prefijo.
  - Port forwarding: en UTM solo existe en el modo de red "Emulated VLAN" (SLIRP), donde la puerta NAT también es `10.0.2.2`, así que la explicación del paso 4 del Lab 1.2 es idéntica.
  - SELinux y firewalld son idénticos en aarch64. En los AVC, `arch=` y `syscall=` muestran otros valores (bind es 200 en aarch64 y 49 en x86_64): irrelevante para el diagnóstico.
  - OpenSCAP: el mismo `ssg-rhel9-ds.xml` sirve; algunas reglas de GRUB/BIOS aparecen como `notapplicable` en aarch64. El escaneo tarda algo más en la VM emulada de UTM si no es aarch64 nativo.
  - `sesearch` (setools-console) y `audit2allow` funcionan igual en ambas.

- **Preguntas probables y respuesta corta:**
  - *"¿Y iptables?"* En RHEL 9 el kernel usa nftables; `iptables` existe solo como capa de compatibilidad (`iptables-nft`). No se mezclan reglas manuales con firewalld: el siguiente `--reload` las pisa.
  - *"¿SELinux frena un exploit de kernel?"* No. Frena lo que el proceso comprometido intenta hacer **después** (leer shadow, escribir en /home, conectarse afuera). Es contención, no prevención.
  - *"¿Puedo apagar SELinux mientras migramos una aplicación?"* No: `semanage permissive -a dominio_t` deja el resto protegido y sigue registrando; después se corrige con fcontext/port/booleanos y se quita.
  - *"¿Diferencia entre `setenforce 0` y `disabled`?"* `setenforce 0` mantiene la política cargada y registrando; `disabled` no carga política y al volver hay que reetiquetar todo (`/.autorelabel`, minutos).
  - *"¿Rich rules o zonas?"* Zonas + `--add-source` resuelven el 90 %. Rich rules cuando se necesita log, límite de tasa, o una excepción puntual (una IP concreta).
  - *"¿firewalld filtra la salida?"* Por defecto no (todo egress permitido). Se puede con *policies* (`firewall-cmd --permanent --new-policy`, firewalld ≥ 1.0, el que trae RHEL 9), fuera del alcance de hoy.
  - *"¿Por qué el 8080 sí arranca y el 82 no?"* Porque 8080 es `http_cache_port_t` y la política permite a `httpd_t` usarlo; 82 es `reserved_port_t`. Lo importante es leer `semanage port -l`, no memorizar.
  - *"¿fail2ban en RHEL?"* Está en EPEL. faillock bloquea la **cuenta**; fail2ban bloquea la **IP** en el firewall. Con llaves SSH y `MaxAuthTries 3` la fuerza bruta ya es inútil; fail2ban reduce ruido en logs.
  - *"¿dnf-automatic reinicia el servidor?"* No. `dnf needs-restarting -r` indica si un kernel o glibc nuevo lo requiere; el reinicio se planifica.
  - *"¿Qué pasa con SELinux al restaurar de backup con `tar`?"* `tar` conserva contextos solo con `--selinux`/`--xattrs`; después de restaurar en un servidor RHEL, `restorecon -Rv` sobre lo restaurado. Es el mismo principio que `cp -a`.

- **Relación con el examen RHCSA (EX200), sección "Manage security":**
  - Configure firewall settings using `firewall-cmd`/firewalld (Bloque 1 completo, incluido `--permanent`/`--reload` y zonas por origen).
  - Manage default file permissions (`umask`, Lab 4.3).
  - Configure key-based authentication for SSH (verificado en Lab 4.1; se configuró el Día 05).
  - Set enforcing and permissive modes for SELinux (Lab 2.1, `/etc/selinux/config`).
  - List and identify SELinux file and process context (`ls -Z`, `ps -eZ`, Lab 2.1).
  - Restore default file contexts (`restorecon`, Lab 2.2).
  - Manage SELinux port labels (`semanage port`, Lab 3.1).
  - Use boolean settings to modify system SELinux settings (`setsebool -P`, Lab 3.3).
  - Diagnose and address routine SELinux policy violations (`ausearch`, `sealert`, Lab 3.4 y el Reto).
  - Regla de oro para el examen: **nunca** `setenforce 0` ni `SELINUX=disabled`; el examen se corrige con el sistema reiniciado, así que todo debe ser `--permanent`, `-P` y con `semanage` (persistente), y hay que verificar después de reiniciar.

---

## Tarea y preparación para el día siguiente

1. **Devolver Apache al estado estándar** para el Día 09 (servicios de red: HTTP, NFS, SMB, FTP), conservando lo aprendido:

```bash
sudo sed -i 's/^Listen .*/Listen 80/' /etc/httpd/conf/httpd.conf
sudo sed -i 's#^DocumentRoot .*#DocumentRoot "/var/www/html"#' /etc/httpd/conf/httpd.conf
sudo sed -i -E 's#^<Directory "(/web|/sitio)">#<Directory "/var/www/html">#' /etc/httpd/conf/httpd.conf
sudo apachectl configtest && sudo systemctl restart httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --zone=internal --add-service=http
for p in 82 8082; do
  sudo firewall-cmd --permanent --remove-port=$p/tcp
  sudo firewall-cmd --permanent --zone=internal --remove-port=$p/tcp
done
# Dejar la red de administración donde debe estar (el reto la había mandado a `drop`)
sudo firewall-cmd --permanent --zone=drop --remove-source=192.168.56.0/24
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
curl -s -o /dev/null -w '%{http_code}\n' http://localhost
```

Salida esperada del final: `internal` con `sources: 192.168.56.0/24`, `public` con las interfaces, y `200`.

(Los `--remove-port` y el `--remove-source` pueden responder `Warning: NOT_ENABLED`, y el `--add-source`/`--add-service` `Warning: ALREADY_ENABLED`, si ya se corrigió durante el reto: son avisos, no errores; el comando devuelve `success` igual.) Las reglas `semanage port` (82, 8082) y `semanage fcontext` (`/web`, `/sitio`) pueden quedarse: no molestan y son un buen recordatorio (`semanage port -l -C`, `semanage fcontext -l -C`).

2. **Mantener el hardening de SSH.** Si el Día 09 necesita entrar por SSH con otro usuario o con contraseña, el único archivo a tocar es `/etc/ssh/sshd_config.d/50-hardening.conf` (agregar el usuario a `AllowUsers` y `sudo systemctl reload sshd`).

3. **Snapshot `dia08-fin`** con la VM apagada o en pausa, después de verificar:

```bash
getenforce; sudo firewall-cmd --list-all | grep services; sudo sshd -T | grep -E '^permitrootlogin'
```

4. **Practicar 20 minutos** (sin mirar el material): restaurar `dia08-pre-reto`, ejecutar de nuevo `romper-dia08.sh` y resolver el ticket cronometrado. Meta: menos de 10 minutos. Quien termine, repetir el Lab 4.4 completo y leer tres reglas en `fail` del reporte.

5. **Lecturas cortas:** `man semanage-fcontext` (sección EXAMPLE), `man firewalld.richlanguage` (primeras dos pantallas), y el capítulo "Using SELinux" de la documentación de RHEL 9 (secciones "Changing SELinux states and modes" y "Troubleshooting problems related to SELinux").

6. **Para el Día 09** no hacen falta discos ni adaptadores nuevos. Conviene dejar descargados: `sudo dnf install -y --downloadonly nfs-utils samba samba-client cifs-utils vsftpd`.
