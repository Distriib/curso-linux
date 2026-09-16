# Día 05 — Redes: NetworkManager, DNS, SSH y troubleshooting

> Al terminar el día, el participante inspecciona la red de un servidor RHEL 9, configura una IP estática con `nmcli`, entiende cómo se resuelven los nombres, administra el acceso remoto con claves SSH y diagnostica una falla de red capa por capa hasta dejarla como estaba.

**Ficha técnica cubierta:**
- RH124 M5 Redes Básicas: Configuración IP, DNS, SSH
- RH134 M3 Redes Avanzadas: NetworkManager, Configuración avanzada IP, Troubleshooting

**Requisitos previos:**
- Snapshot `dia04-fin` tomado y la VM arrancando sin errores.
- La VM registrada y con repositorios: `sudo dnf repolist` muestra `rhel-9-for-x86_64-baseos-rpms` y `...-appstream-rpms` (en UTM, `aarch64`). Hoy se instalan `bind-utils`, `traceroute`, `tcpdump` y `rsync` (y, si están disponibles y no vinieran ya en la instalación, `ipcalc`, `NetworkManager-tui` y `mtr`, que son opcionales).
- Acceso por SSH desde el equipo propio funcionando: `ssh -p 2222 student@localhost`.
- Estructura `~/empresa` del Día 2. Si no existe: `mkdir -p ~/empresa/{documentos,clientes,backups,logs} && touch ~/empresa/documentos/doc{1..5}.txt`.
- Hipervisor a mano: en el Bloque 3 se apaga la VM unos minutos para añadir un segundo adaptador de red (Host-only). En VirtualBox debe existir una red host-only (File > Tools > Network Manager); en UTM se usa el modo "Host Only".
- En Windows: cliente OpenSSH disponible en PowerShell (`ssh -V` responde `OpenSSH_for_Windows_...`). En macOS y Linux viene incluido.
- Del Día 4 quedan `httpd` instalado y habilitado (aparecerá escuchando en el puerto 80 al inspeccionar los sockets) y el firewall todavía sin abrir: en la zona `public` solo están `ssh`, `cockpit` y `dhcpv6-client`. Ambas cosas se usan hoy como ejemplo; el firewall se trabaja el Día 8.

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:05 | 5 | Repaso | Tres preguntas del Día 4 (servicios, journalctl, procesos) con la terminal abierta |
| 0:05–0:35 | 30 | Bloque 1 | Conceptos de red: IP, máscara, gateway, DNS, puertos, IPv6, NAT vs Host-only vs Bridged. Lab 1.1 con `ipcalc` |
| 0:35–1:10 | 35 | Bloque 2 | Inspección: `ip`, `ss`, `ping`, `tracepath`, `hostnamectl`, `nmcli` de lectura. Lab 2.1 inspección, Lab 2.2 DNS con `dig` |
| 1:10–2:05 | 55 | Bloque 3 | NetworkManager: keyfiles, añadir adaptador Host-only, perfil `lab` con IP estática, modificar (DNS, dos IPs, rutas, MTU, IPv6), nmtui, `/etc/hosts`, hostname FQDN |
| 2:05–2:20 | 15 | Descanso | |
| 2:20–3:05 | 45 | Bloque 4 | SSH: sshd y `sshd_config.d/`, cliente, claves ed25519, `ssh-copy-id`, agente, `~/.ssh/config`, `known_hosts`, `scp`/`sftp`/`rsync`, túnel local |
| 3:05–3:30 | 25 | Bloque 5 | Troubleshooting por capas, tabla síntoma → capa → comando, `journalctl -u NetworkManager`, `tcpdump` |
| 3:30–3:50 | 20 | Reto individual | `romper-red.sh`: diagnosticar y reparar documentando qué comando delató cada falla |
| 3:50–4:00 | 10 | Cierre | Resumen, cheatsheet, snapshot `dia05-fin`, tarea (dos discos de 5 GB para el Día 6) |

## Prioridad si falta tiempo

**Imprescindible**
- Leer la configuración actual: `ip -br a`, `ip route`, `cat /etc/resolv.conf`, `nmcli device status`, `nmcli connection show`, `ss -tulpn`.
- Añadir el adaptador Host-only y crear el perfil `lab` con IP estática usando `nmcli con add ... ipv4.method manual`; verificar con `ip -br a` y conectarse desde el host directo a la IP.
- Saber que `nmcli con mod` no aplica hasta `nmcli con up`, y que `/etc/resolv.conf` lo escribe NetworkManager.
- Claves SSH: `ssh-keygen -t ed25519`, `ssh-copy-id` (o su equivalente en Windows), permisos de `~/.ssh`.
- El método de diagnóstico por capas (link → IP → gateway → internet → DNS → servicio → firewall) y el reto.

**Importante**
- DNS con `dig` (`+short`, `@servidor`, `-x`) y `getent hosts`; `/etc/hosts` para nombres locales; `hostnamectl set-hostname` con FQDN.
- Modificaciones del perfil: DNS, dos IPs, ruta estática, `nmcli con reload`, `nmcli device reapply`, ver el keyfile en `/etc/NetworkManager/system-connections/`.
- `~/.ssh/config` con alias, `known_hosts` y `ssh-keygen -R`, `scp` y `rsync -avz --delete`.
- `tcpdump -i <if> -n icmp` mientras se hace ping.

**Si sobra tiempo**
- `nmtui`, MTU, `ipv6.method disabled` / IPv6 estático, `ipv4.never-default`.
- `sftp` interactivo, `ssh-agent`/`ssh-add`, túnel local `ssh -L`.
- Mirar (sin cambiar) la configuración del servidor SSH: `ls /etc/ssh/sshd_config.d/` y `sudo sshd -T | grep -iE 'permitrootlogin|passwordauthentication|pubkeyauthentication'`. El hardening se hace el Día 8.
- `traceroute`/`mtr`, `ip -s link`, `ip neigh`, `nmcli -f all device show`.
- Mención de bonding y VLAN (existencia, no práctica).

---

## Bloque 1 — Conceptos de red que todo administrador debe tener claros

### Conceptos (25 min)

Antes de tocar un solo comando conviene fijar cinco ideas. Todas se van a ver en la terminal en el bloque siguiente, así que aquí se explican con analogías y se anotan los números que luego aparecerán en pantalla.

**1. Dirección IPv4, máscara y CIDR.** Una dirección IPv4 son cuatro números de 0 a 255 separados por puntos (`192.168.56.10`). Sola no dice nada: hace falta la máscara para saber qué parte identifica al "barrio" (la red) y qué parte a la "casa" (el host). La forma corta de escribir la máscara es CIDR: `/24` significa que los primeros 24 bits (los tres primeros octetos) son la red. Entonces:

- `192.168.56.10/24` → red `192.168.56.0`, hosts válidos `.1` a `.254`, broadcast `192.168.56.255`.
- `10.0.2.15/24` → red `10.0.2.0`, broadcast `10.0.2.255`. Esa es la que ven todas las VM en NAT de VirtualBox y de UTM en modo "Emulated VLAN".
- Máscara `255.255.255.0` es exactamente lo mismo que `/24`. `/16` = `255.255.0.0`, `/8` = `255.0.0.0`.

Decir en clase: "Dos equipos se hablan directamente solo si están en la misma red. Si no, necesitan un intermediario: el gateway."

**2. Gateway y ruta por defecto.** El gateway es el router de la red local: la "puerta del edificio". El sistema tiene una tabla de rutas (`ip route`) que dice por dónde salir hacia cada destino. La línea `default via 10.0.2.2` es la ruta por defecto: "todo lo que no sepa a dónde va, se lo entrego a 10.0.2.2". Sin ruta por defecto no hay internet; con una ruta por defecto equivocada, tampoco. Una red puede tener rutas estáticas adicionales ("para llegar a 10.10.0.0/24 usa el router 192.168.56.1").

**3. DNS y el orden de resolución.** Las personas usan nombres (`redhat.com`); las máquinas usan IPs. Traducir uno en otro es resolver. En Linux el orden lo dicta `/etc/nsswitch.conf`, línea `hosts:`; en RHEL 9 es `files dns myhostname`:

1. `files` → `/etc/hosts`: un archivo local con líneas `IP nombre`. Gana siempre. Sirve para nombres internos que no están en ningún DNS.
2. `dns` → los servidores listados en `/etc/resolv.conf` (`nameserver 10.0.2.3`). En RHEL 9 ese archivo **lo escribe NetworkManager**: si se edita a mano se pierde en la siguiente activación de la conexión.
3. `myhostname` → resuelve el propio hostname aunque no esté en `/etc/hosts`.

Analogía: `/etc/hosts` es la agenda del teléfono; el DNS es el directorio público.

**4. Puertos, sockets, TCP y UDP.** La IP identifica la máquina; el puerto (0–65535) identifica el servicio dentro de la máquina: 22 SSH, 80 HTTP, 443 HTTPS, 53 DNS, 2049 NFS, 445 SMB. Un socket es la combinación `IP:puerto` en la que un programa escucha o por la que conversa. TCP establece una conexión confiable (SSH, HTTP); UDP envía sin conexión y sin garantía (DNS, NTP). El comando `ss -tulpn` muestra quién escucha en qué puerto: es la radiografía de los servicios publicados de un servidor.

**5. IPv6 en dos frases.** Toda interfaz activa tiene al menos una dirección IPv6 *link-local* que empieza por `fe80::` y solo sirve dentro del mismo segmento; se ve en `ip a` como `inet6 fe80::.../64 scope link`. Configurar IPv6 estático es igual que IPv4 con `nmcli` (`ipv6.method manual ipv6.addresses ...`), y es objetivo del RHCSA, así que se practica al final del Bloque 3.

**Los tres modos de red del hipervisor.** Esto explica por qué la VM tiene la IP que tiene y por qué el curso usa dos adaptadores:

| Modo | Qué IP recibe la VM | ¿La VM sale a internet? | ¿El host llega a la VM? | ¿Otras PCs de la oficina llegan? |
|---|---|---|---|---|
| NAT (VirtualBox) / Emulated VLAN (UTM) | `10.0.2.15/24`, gateway `10.0.2.2`, DNS `10.0.2.3` (red privada del hipervisor, igual en todas las VMs) | Sí | Solo con port forwarding (2222→22, 8080→80) | No |
| Host-only | Red privada entre host y VM (`192.168.56.0/24` en VirtualBox; en UTM la asigna macOS, p. ej. `192.168.64.x`) | No (no hay gateway) | Sí, directo por IP | No |
| Bridged | Una IP de la red física real (la del DHCP de la oficina o de la casa) | Sí | Sí | Sí |

El curso usa **NAT + Host-only**: NAT garantiza internet sin depender de la red del participante (Wi-Fi de casa, red institucional con filtros); Host-only da una IP fija y estable para practicar configuración estática, SSH directo y servicios cliente-servidor sin port forwarding. Bridged sería lo más parecido a producción, pero en redes corporativas suele estar prohibido o bloqueado por DHCP.

### Lab 1.1 — Calcular red y broadcast con ipcalc (5 min)

- **Objetivo:** comprobar mentalmente y con la herramienta a qué red pertenece una IP dada una máscara.

1. Verificar que `ipcalc` está instalado (en RHEL 9 suele venir con la instalación Server; si no, se instala. ⚠️ Verificar en la VM antes de la clase):

```bash
rpm -q ipcalc || sudo dnf install -y ipcalc
```

Salida esperada:
```
ipcalc-1.0.0-5.el9.x86_64
```

2. Calcular la red de la IP host-only que se va a configurar hoy:

```bash
ipcalc -bmn 192.168.56.10/24
```

Salida esperada (el orden de las líneas puede variar según la versión de `ipcalc`; lo importante son los valores):
```
NETMASK=255.255.255.0
BROADCAST=192.168.56.255
NETWORK=192.168.56.0
```

3. Repetir con la IP NAT y con una máscara distinta para ver el efecto:

```bash
ipcalc -bmn 10.0.2.15/24
ipcalc -bmn 172.16.40.130/26
```

Salida esperada (segunda):
```
NETMASK=255.255.255.192
BROADCAST=172.16.40.191
NETWORK=172.16.40.128
```

Qué observar: con `/26` la red tiene solo 62 hosts útiles (`.129`–`.190`). Preguntar: "¿`172.16.40.200/26` está en la misma red que `.130`?" (No: pertenece a `172.16.40.192/26`).

- **Checkpoint:** pegar en el chat la salida de `ipcalc -bmn 192.168.56.10/24`.

---

## Bloque 2 — Inspección: leer la red antes de tocarla

### Conceptos (5 min)

La regla del día: **primero mirar, después cambiar**. En RHEL 9 las herramientas modernas son `ip` (reemplaza a `ifconfig` y `route`) y `ss` (reemplaza a `netstat`); `ifconfig` y `netstat` ni siquiera están instalados (paquete `net-tools`) y no hay que acostumbrarse a ellos. Para la capa de NetworkManager se usa `nmcli`, que en modo lectura no necesita `sudo`. Toda la información que se va a leer ahora es la que un ticket de "no tengo red" obliga a revisar.

### Lab 2.1 — Inspección completa de la VM (18 min)

- **Objetivo:** identificar interfaces, IPs, rutas, vecinos, sockets abiertos y hostname de la VM, y saber qué comando da cada dato.

1. Interfaces y direcciones, versión larga y versión breve:

```bash
ip addr
ip -br a
```

Salida esperada (breve; en UTM la interfaz se llamará `enp0s1` o similar):
```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 fe80::a00:27ff:fe4c:9b1e/64
```

Qué observar: `lo` es el loopback (siempre `127.0.0.1`); `enp0s3` está `UP` con IPv4 `10.0.2.15/24` y una IPv6 link-local `fe80::`. En la salida larga aparecen además la MAC (`link/ether 08:00:27:...` en VirtualBox), el MTU `1500` y `scope global dynamic` (la IP vino por DHCP).

2. Estado del enlace y estadísticas:

```bash
ip link
ip -s link show enp0s3
```

Salida esperada (resumida):
```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:4c:9b:1e brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped  missed   mcast
       1843215   4021      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
        302118   2210      0       0       0       0
```

Qué observar: `LOWER_UP` = hay cable (portadora). Si un día aparece `NO-CARRIER` o `state DOWN`, el problema es físico o del hipervisor, no de configuración. Contadores `errors`/`dropped` distintos de cero delatan problemas de capa física o saturación.

3. Tabla de rutas:

```bash
ip route
ip -4 route
```

Salida esperada:
```
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
```

Qué observar: la primera línea es la ruta por defecto (gateway `10.0.2.2`, aprendida por DHCP). La segunda la crea el kernel al asignar la IP: "a `10.0.2.0/24` llego directo por `enp0s3`". `ip -4 route` filtra solo IPv4 (`ip -6 route` muestra las de IPv6). Nota: el modo breve `-br` existe para `ip a`, `ip link` e `ip neigh`, **no** para `ip route`: aquí se filtra por familia, no por formato.

4. Vecinos (tabla ARP) y conectividad básica:

```bash
ping -c 3 10.0.2.2
ip neigh
ping -c 3 8.8.8.8
```

Salida esperada:
```
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
10.0.2.2 dev enp0s3 lladdr 52:54:00:12:35:02 REACHABLE
3 packets transmitted, 3 received, 0% packet loss
```

Qué observar: `ip neigh` muestra la MAC del gateway una vez que se le ha hablado. Si el gateway aparece como `FAILED` o `INCOMPLETE`, la VM no consigue ni siquiera hablar con su router.

5. Camino hacia un destino externo:

```bash
tracepath -n 8.8.8.8
sudo dnf install -y traceroute
traceroute -n 8.8.8.8
```

(`tracepath` viene en `iputils`, ya instalado; `traceroute` hay que instalarlo. `mtr` es opcional: `sudo dnf install -y mtr` — ⚠️ Verificar en la VM antes de la clase que `mtr` esté en los repositorios habilitados; si no lo estuviera, el día funciona igual sin él.)

Salida esperada (en NAT del hipervisor):
```
 1?: [LOCALHOST]                      pmtu 1500
 1:  10.0.2.2                                              0.422ms
 2:  no reply
 3:  no reply
...
```

Qué observar: es normal que en NAT solo responda el primer salto (`10.0.2.2`): el motor NAT del hipervisor no reenvía los mensajes ICMP "TTL exceeded" de los routers externos (⚠️ Verificar en la VM antes de la clase: en UTM/Emulated VLAN puede que ni siquiera responda el salto 1). En un servidor con red real se ven todos los saltos. `mtr -n -c 5 8.8.8.8` combina ping y traceroute en vivo; se muestra rápido y se sigue.

6. Sockets que escuchan y conexiones establecidas:

```bash
sudo ss -tulpn
ss -tan
```

Salida esperada (primera, resumida):
```
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0          127.0.0.1:323        0.0.0.0:*     users:(("chronyd",pid=712,fd=5))
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=890,fd=3))
tcp   LISTEN 0      128             [::]:22            [::]:*     users:(("sshd",pid=890,fd=4))
tcp   LISTEN 0      511                *:80               *:*     users:(("httpd",pid=1204,fd=4),...)
```

Qué observar: `-t` TCP, `-u` UDP, `-l` solo los que escuchan, `-p` proceso (necesita `sudo` para ver los de otros usuarios), `-n` números en vez de nombres. La salida real puede traer más líneas (`[::1]:323` de chronyd, `rpcbind` en el 111, `cockpit` en el 9090 si se habilitó el Día 4): lo mínimo es `sshd`, `chronyd` y el `httpd` que quedó instalado y habilitado el Día 4 (escucha en `*:80`, es decir en IPv4 e IPv6 a la vez, aunque el firewall todavía no lo deje entrar desde fuera). `0.0.0.0:22` = sshd escucha en todas las IPv4. En `ss -tan` se ve la propia sesión SSH como `ESTAB 10.0.2.15:22 10.0.2.2:xxxxx`: el origen es `10.0.2.2` porque el port forwarding la hace llegar desde el hipervisor.

7. Hostname y archivos de resolución:

```bash
hostname
hostnamectl
cat /etc/hosts
cat /etc/resolv.conf
grep ^hosts: /etc/nsswitch.conf
```

Salida esperada (resumida):
```
rhel01
 Static hostname: rhel01
       Icon name: computer-vm
         Chassis: vm
  Virtualization: oracle
Operating System: Red Hat Enterprise Linux 9.6 (Plow)
          Kernel: Linux 5.14.0-570.12.1.el9_6.x86_64
    Architecture: x86-64
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# Generated by NetworkManager
nameserver 10.0.2.3
hosts:      files dns myhostname
```

Qué observar: la primera línea de `/etc/resolv.conf` dice quién lo genera. La versión menor (9.4, 9.5, 9.6...) y el kernel dependen de la ISO usada el Día 1. En UTM `Virtualization` dirá `qemu` o `apple` y `Architecture: arm64`.

8. La vista de NetworkManager:

```bash
nmcli general
nmcli device status
nmcli connection show
nmcli device show enp0s3
```

Salida esperada (resumida):
```
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
connected  full          enabled  enabled  enabled  enabled

DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
lo      loopback  connected (externally)  lo

NAME    UUID                                  TYPE      DEVICE
enp0s3  5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  ethernet  enp0s3
lo      2a1c...                               loopback  lo

GENERAL.DEVICE:                         enp0s3
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         08:00:27:4C:9B:1E
GENERAL.MTU:                            1500
GENERAL.STATE:                          100 (connected)
GENERAL.CONNECTION:                     enp0s3
WIRED-PROPERTIES.CARRIER:               on
IP4.ADDRESS[1]:                         10.0.2.15/24
IP4.GATEWAY:                            10.0.2.2
IP4.ROUTE[1]:                           dst = 10.0.2.0/24, nh = 0.0.0.0, mt = 100
IP4.ROUTE[2]:                           dst = 0.0.0.0/0, nh = 10.0.2.2, mt = 100
IP4.DNS[1]:                             10.0.2.3
IP6.ADDRESS[1]:                         fe80::a00:27ff:fe4c:9b1e/64
```

Qué observar y decir en clase: NetworkManager distingue **device** (la tarjeta física, `enp0s3`) de **connection** (el perfil de configuración, que aquí también se llama `enp0s3` porque el instalador lo nombró igual). La línea `lo loopback connected (externally)` y el perfil `lo` aparecen desde NetworkManager 1.42 (RHEL 9.2+); en 9.0/9.1 no se listan y no pasa nada. Un dispositivo puede tener varios perfiles, pero solo uno activo. Esa distinción es la clave para entender todo el Bloque 3.

- **Checkpoint:** pegar en el chat la salida de `ip -br a && ip route && nmcli device status`.

### Lab 2.2 — Resolución de nombres con dig y getent (12 min)

- **Objetivo:** consultar DNS a mano, distinguir la respuesta del DNS de la del sistema (`getent`) y entender qué servidor está contestando.

1. Instalar las utilidades de DNS (paquete `bind-utils`: `dig`, `nslookup`, `host`):

```bash
sudo dnf install -y bind-utils
```

Salida esperada:
```
Installed:
  bind-libs-32:9.16.23-24.el9.x86_64   bind-license-32:9.16.23-24.el9.noarch   bind-utils-32:9.16.23-24.el9.x86_64
Complete!
```

(La versión exacta `9.16.23-NN.el9` depende de la 9.x instalada; en UTM termina en `.aarch64`.)

2. Consulta completa y consulta corta:

```bash
dig redhat.com
dig +short redhat.com
```

Salida esperada (resumida; la IP cambia con el tiempo):
```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 27541
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;redhat.com.                    IN      A

;; ANSWER SECTION:
redhat.com.             300     IN      A       34.235.198.240

;; Query time: 38 msec
;; SERVER: 10.0.2.3#53(10.0.2.3)

34.235.198.240
```

Qué observar: `status: NOERROR` (existe), `ANSWER SECTION` con el registro `A`, el TTL (`300` segundos de caché) y `SERVER: 10.0.2.3` = el DNS que respondió, el mismo de `/etc/resolv.conf`. Si `status` fuese `NXDOMAIN`, el nombre no existe; si `SERVFAIL`, el servidor no pudo.

3. Preguntar a otro servidor DNS y resolución inversa:

```bash
dig @8.8.8.8 +short redhat.com
dig -x 8.8.8.8 +short
nslookup redhat.com
host redhat.com
```

Salida esperada:
```
34.235.198.240
dns.google.
Server:         10.0.2.3
Address:        10.0.2.3#53
Non-authoritative answer:
Name:   redhat.com
Address: 34.235.198.240
redhat.com has address 34.235.198.240
redhat.com mail is handled by 10 us-smtp-inbound-1.mimecast.com.
```

Qué observar: `@8.8.8.8` salta el DNS configurado. Es la prueba definitiva cuando se sospecha del DNS local: si `@8.8.8.8` responde y sin `@` no, el problema es el resolver de la red, no internet.

4. Lo que ve el sistema (no solo el DNS):

```bash
getent hosts localhost
getent hosts rhel01
getent hosts redhat.com
```

Salida esperada:
```
::1             localhost localhost.localdomain localhost6 localhost6.localdomain6
fe80::a00:27ff:fe4c:9b1e rhel01
34.235.198.240  redhat.com
```

Qué observar: `getent hosts` sigue el orden de `nsswitch.conf`: `localhost` sale de `/etc/hosts`, `rhel01` lo responde `myhostname` (no está en `/etc/hosts`) y `redhat.com` viene del DNS. Detalle: `getent hosts` pregunta primero por IPv6 y solo si no hay respuesta pregunta por IPv4; por eso `rhel01` puede salir con su dirección link-local `fe80::` (o con `10.0.2.15` según la versión; ⚠️ Verificar en la VM antes de la clase) y un dominio con registro AAAA saldría con IPv6. Para forzar IPv4: `getent ahostsv4 rhel01` → `10.0.2.15 STREAM rhel01`. `dig` solo pregunta al DNS; `getent` pregunta como lo haría cualquier programa. Por eso, cuando un servicio "no resuelve", se prueba con `getent`, no solo con `dig`.

5. Demostrar que `/etc/hosts` gana al DNS:

```bash
echo "127.0.0.1 redhat.com" | sudo tee -a /etc/hosts
getent hosts redhat.com
dig +short redhat.com
sudo sed -i '/127.0.0.1 redhat.com/d' /etc/hosts
getent hosts redhat.com
```

Salida esperada:
```
127.0.0.1 redhat.com
127.0.0.1       redhat.com
34.235.198.240
34.235.198.240  redhat.com
```

Qué observar: la primera línea es el eco de `tee`. Con la línea en `/etc/hosts`, el sistema resuelve `127.0.0.1` aunque `dig` siga diciendo la IP real. Es un truco útil (probar una web antes de cambiar el DNS) y también un vector clásico de secuestro de nombres: revisar `/etc/hosts` es parte de cualquier troubleshooting de DNS.

- **Checkpoint:** pegar en el chat la salida de `dig +short redhat.com; getent hosts rhel01; grep ^hosts: /etc/nsswitch.conf`.

---

## Bloque 3 — Configuración con NetworkManager

### Conceptos (10 min)

**NetworkManager (NM) es el único gestor de red en RHEL 9.** Los `network-scripts` de RHEL 7 ya no existen y los archivos `ifcfg-*` de `/etc/sysconfig/network-scripts/` quedaron obsoletos: en RHEL 9 los perfiles se guardan como **keyfiles** en `/etc/NetworkManager/system-connections/NOMBRE.nmconnection`, archivos INI legibles, propiedad de root con permisos `600`. Si un servidor migrado aún tiene `ifcfg-*`, `nmcli connection migrate` los convierte a keyfile. En RHEL 10 ya no hay soporte de `ifcfg`.

Tres formas de configurar, todas equivalentes y todas terminan escribiendo el mismo keyfile:

| Herramienta | Cuándo usarla |
|---|---|
| `nmcli` | Siempre que se pueda: scriptable, copiable en un ticket, es lo que pide el examen |
| `nmtui` | Interfaz de texto con menús; útil para quien está empezando o en la consola de emergencia |
| Editar el `.nmconnection` + `nmcli con reload` | Para plantillas o cuando se copia una configuración entre servidores |

El ciclo mental con `nmcli`:

1. `nmcli con add ...` crea el perfil (escribe el archivo). Si el dispositivo ya tiene otro perfil activo no toca la red; si el dispositivo está libre y `connection.autoconnect` es `yes` (el valor por defecto), NM lo activa de inmediato.
2. `nmcli con mod ...` cambia el perfil **en disco**. **No aplica nada** hasta que se reactive.
3. `nmcli con up NOMBRE` (o `nmcli device reapply IF`) aplica.
4. Verificar con `ip -br a`, `ip route`, `cat /etc/resolv.conf`.

Decir en clase: "`con mod` es editar el archivo; `con up` es reiniciar la interfaz. El error número uno de hoy será modificar y no aplicar."

Propiedades que se usan hoy: `ipv4.method` (`auto` = DHCP, `manual` = estático, `disabled`), `ipv4.addresses` (una o varias `IP/prefijo`), `ipv4.gateway`, `ipv4.dns`, `ipv4.dns-search`, `ipv4.routes`, `ipv4.never-default`, `ipv6.method`, `802-3-ethernet.mtu`, `connection.autoconnect`. El prefijo `+` añade a una lista y `-` quita (`+ipv4.dns 8.8.8.8`). Todo está en `man nm-settings-nmcli` y con ejemplos en `man nmcli-examples`.

Cuando aparece una tarjeta nueva sin perfil, NM crea automáticamente uno llamado `Wired connection 1` con DHCP (vive en `/run/NetworkManager/system-connections/`, no en `/etc`, hasta que alguien lo modifica). Excepción importante: si está instalado el paquete `NetworkManager-config-server` (habitual en instalaciones tipo Server; comprobar con `rpm -q NetworkManager-config-server`), su archivo `/usr/lib/NetworkManager/conf.d/00-server.conf` fija `no-auto-default=*` y NM **no** crea ese perfil: la tarjeta nueva aparece `disconnected` sin IP. En ambos casos hoy se termina con un solo perfil, `lab`. ⚠️ Verificar en la VM antes de la clase cuál de los dos comportamientos tiene la instalación del curso.

### Lab 3.1 — Añadir el adaptador Host-only en el hipervisor (15 min)

- **Objetivo:** que la VM tenga una segunda tarjeta de red conectada a la red host-only y que NM la detecte.

1. Apagar la VM limpiamente (la sesión SSH se cierra; es esperado):

```bash
sudo poweroff
```

2. Añadir el adaptador según el hipervisor.

**VirtualBox 7 (Windows y Mac Intel):**
- Comprobar que existe la red host-only: menú *File > Tools > Network Manager*, pestaña *Host-only Networks*. Debe haber una (en Windows se llama "VirtualBox Host-Only Ethernet Adapter"; en Mac, "HostNetwork" o `vboxnet0`) con IPv4 `192.168.56.1/24`. Si no existe, botón *Create*. El servidor DHCP puede quedar activado (reparte desde `.101`, no choca con la `.10` que se va a usar).
- Seleccionar la VM `rhel01` > *Settings > Network > Adapter 2*: marcar *Enable Network Adapter*, *Attached to: Host-only Adapter*, *Name:* la red anterior. Dejar *Cable Connected* marcado. OK.
- Alternativa por línea de comandos en el host (con la VM apagada):

```bash
VBoxManage modifyvm "rhel01" --nic2 hostonly --host-only-adapter2 "VirtualBox Host-Only Ethernet Adapter"
```

(En macOS el nombre del adaptador es el que muestra `VBoxManage list hostonlyifs`. En VirtualBox 7 la opción se escribe `--host-only-adapter2`; en versiones anteriores era `--hostonlyadapter2`, y VirtualBox 7 acepta ambas. En VirtualBox no se puede habilitar un adaptador nuevo con la VM encendida; sí se puede cambiar el modo de uno ya habilitado.)

**UTM (Mac Apple Silicon, demo del instructor):**
- Con la VM apagada, abrir la configuración de la VM (icono de controles), en la lista de dispositivos pulsar *New...* > *Network*. *Network Mode: Host Only*, *Emulated Network Card: virtio-net-pci*. *Save*.
- El primer adaptador debe seguir en modo *Emulated VLAN* (es el que tiene el port forwarding 2222→22 y 8080→80).
- La red host-only de UTM la crea macOS con el framework vmnet y **el rango lo decide macOS**, no el usuario. Para leerlo, en la terminal del Mac una vez arrancada la VM:

```bash
ifconfig | grep -A 4 '^bridge1'
```

Salida esperada (ejemplo; puede ser otra red):
```
bridge100: flags=8a63<UP,BROADCAST,SMART,RUNNING,ALLMULTI,SIMPLEX,MULTICAST> mtu 1500
	...
	inet 192.168.64.1 netmask 0xffffff00 broadcast 192.168.64.255
```

Esa `inet` es la IP del Mac en la red host-only; la VM debe usar una IP de esa misma red (por ejemplo `192.168.64.10/24`). En todo el bloque, donde diga `192.168.56.10` el instructor usa la de su rango. Otra pista: macOS reparte DHCP en esa red, así que la IP que NM obtenga automáticamente en la tarjeta nueva también revela la red.

3. Encender la VM, volver a conectarse por SSH y detectar la tarjeta nueva:

```bash
ssh -p 2222 student@localhost
nmcli device status
ip -br a
```

Salida esperada (VirtualBox):
```
DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
enp0s8  ethernet  connected               Wired connection 1
lo      loopback  connected (externally)  lo

lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             10.0.2.15/24 fe80::a00:27ff:fe4c:9b1e/64
enp0s8           UP             192.168.56.101/24 fe80::5c8e:2b1d:7a40:9c3f/64
```

Salida esperada si la instalación tiene `NetworkManager-config-server` (`no-auto-default=*`):
```
DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
enp0s8  ethernet  disconnected            --
lo      loopback  connected (externally)  lo
```

Qué observar: apareció `enp0s8` (en UTM probablemente `enp0s2`; **verificar con `nmcli device`**). Sin `config-server`, NM le creó solo el perfil `Wired connection 1` con DHCP: si el DHCP del host-only está activo tendrá una IP `.10x`; si no, aparecerá `connecting (getting IP configuration)` y luego sin IPv4. Con `config-server`, la tarjeta queda `disconnected` y sin perfil. En cualquier caso, el siguiente lab la deja con el perfil `lab`. Anotar el nombre de la interfaz: se usa en todos los comandos siguientes. La dirección `fe80::` de `enp0s8` suele ser aleatoria (NM genera las link-local en modo `stable-privacy` para los perfiles nuevos), a diferencia de la de `enp0s3`, cuyo perfil de instalación usa `eui64` (derivada de la MAC).

4. Confirmar que la ruta por defecto sigue siendo la de NAT (el host-only no debe tomarla):

```bash
ip route
```

Salida esperada:
```
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.101 metric 101
```

(La tercera línea solo aparece si NM creó `Wired connection 1` y obtuvo IP por DHCP; con `config-server` no hay ruta para `enp0s8` todavía.)

- **Checkpoint:** pegar en el chat la salida de `nmcli device status` mostrando las dos tarjetas ethernet.

### Lab 3.2 — Crear el perfil `lab` con IP estática (12 min)

- **Objetivo:** dejar `enp0s8` con `192.168.56.10/24` fija, persistente, sin gateway, y entrar desde el host directo a esa IP sin port forwarding.

1. Ver cómo es un keyfile antes de crear el nuestro:

```bash
sudo ls -l /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/enp0s3.nmconnection
```

Salida esperada:
```
-rw-------. 1 root root 262 ... enp0s3.nmconnection
[connection]
id=enp0s3
uuid=5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03
type=ethernet
autoconnect-priority=-999
interface-name=enp0s3
timestamp=1725300000

[ethernet]

[ipv4]
method=auto

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
```

Qué observar: cada sección corresponde a un grupo de propiedades de `nmcli` (`[ipv4] method=auto` ↔ `ipv4.method auto`). Permisos `600` de root: contiene potencialmente contraseñas Wi-Fi o VPN. En `/etc` solo está `enp0s3.nmconnection`: el perfil automático `Wired connection 1` (si existe) vive en `/run/NetworkManager/system-connections/` y se ve con `nmcli -f NAME,FILENAME connection show`.

2. Crear el perfil. Un solo comando, en una línea (sustituir `enp0s8` por la interfaz detectada y, en UTM, la IP por una del rango host-only del Mac):

```bash
sudo nmcli connection add type ethernet con-name lab ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.56.10/24
```

Salida esperada:
```
Connection 'lab' (9c2f7f1e-3d1a-4b0e-9d0c-6a1b2c3d4e5f) successfully added.
```

Qué observar: **no se pone `ipv4.gateway`**. Una red host-only no tiene salida; si se le pone gateway, aparece una segunda ruta por defecto y la VM puede intentar salir a internet por donde no hay nada. Si `enp0s8` estaba `disconnected` (caso `config-server`), NM activa `lab` en cuanto se crea, porque el dispositivo está libre y `autoconnect` es `yes`; el `con up` del paso siguiente no hace daño.

3. Activar y verificar:

```bash
sudo nmcli connection up lab
ip -br a
ip route
nmcli device status
```

Salida esperada:
```
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
enp0s8           UP             192.168.56.10/24 fe80::5c8e:2b1d:7a40:9c3f/64
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.10 metric 101
DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
enp0s8  ethernet  connected               lab
lo      loopback  connected (externally)  lo
```

4. Eliminar el perfil automático para que no compita en el próximo arranque, y ver el archivo creado:

```bash
sudo nmcli connection delete "Wired connection 1"
nmcli connection show
sudo cat /etc/NetworkManager/system-connections/lab.nmconnection
```

Salida esperada:
```
Connection 'Wired connection 1' (uuid) successfully deleted.
NAME    UUID                                  TYPE      DEVICE
enp0s3  5fb06bd0-...                          ethernet  enp0s3
lab     9c2f7f1e-...                          ethernet  enp0s8
lo      2a1c...                               loopback  lo
[connection]
id=lab
uuid=9c2f7f1e-3d1a-4b0e-9d0c-6a1b2c3d4e5f
type=ethernet
interface-name=enp0s8

[ethernet]

[ipv4]
address1=192.168.56.10/24
method=manual

[ipv6]
addr-gen-mode=default
method=auto

[proxy]
```

Si `delete` responde `Error: unknown connection 'Wired connection 1'`, es que NM no lo había creado (por ejemplo, porque el DHCP del host-only estaba apagado y la tarjeta nunca se activó): no pasa nada.

5. Desde el **equipo propio** (PowerShell en Windows, Terminal en Mac), conectarse directo a la IP, sin `-p 2222`:

```bash
ping -c 2 192.168.56.10      # en Windows: ping -n 2 192.168.56.10
ssh student@192.168.56.10
hostname
exit
```

Salida esperada:
```
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:....
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
student@192.168.56.10's password:
[student@rhel01 ~]$ hostname
rhel01
```

Qué observar: es la primera vez que se entra a la VM "como a un servidor de verdad", por su IP y su puerto 22, sin la traducción del hipervisor. El firewall de RHEL ya permite `ssh` en la zona `public`, por eso funciona de inmediato. Desde esta IP se probarán NFS y SMB en los próximos días.

6. Comprobar persistencia con un reinicio de la conexión (un reinicio completo de la VM se hace al cierre). **Hacerlo desde la sesión por port forwarding (`ssh -p 2222`)**: una sesión abierta hacia `192.168.56.10` se corta al bajar `lab`:

```bash
sudo nmcli connection down lab && sudo nmcli connection up lab
ip -br a show enp0s8
```

Salida esperada:
```
Connection 'lab' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
enp0s8           UP             192.168.56.10/24 fe80::5c8e:2b1d:7a40:9c3f/64
```

- **Checkpoint:** pegar en el chat la salida de `nmcli -g ipv4.method,ipv4.addresses,ipv4.gateway connection show lab` (esperado: `manual`, `192.168.56.10/24` y gateway vacío).

### Lab 3.3 — Modificar el perfil: DNS, varias IPs, rutas, hostname, /etc/hosts, MTU, IPv6 (18 min)

- **Objetivo:** practicar `nmcli con mod` con las propiedades que pide el RHCSA y comprobar cada cambio en el sistema. Todo el lab se hace desde la sesión `ssh -p 2222 student@localhost` (NAT): cada `nmcli con up lab` reinicia la interfaz host-only y cortaría una sesión abierta a `192.168.56.10`.
- **Ritmo:** los pasos 1–4 (DNS, dos IPs, ruta, hostname y `/etc/hosts`) son el núcleo y deben caber en ~12 min con todos ejecutando. Los pasos 5–7 (MTU, IPv6, `reload`/`nmtui`) están en "Si sobra tiempo": si el reloj aprieta, el instructor los muestra en su VM y los participantes los hacen en la tarea.

1. DNS y dominio de búsqueda en el perfil `lab`, y ver que `resolv.conf` cambia solo:

```bash
sudo nmcli connection modify lab ipv4.dns 1.1.1.1 +ipv4.dns 8.8.8.8 ipv4.dns-search lab.local
cat /etc/resolv.conf
sudo nmcli connection up lab
cat /etc/resolv.conf
```

Salida esperada (antes y después de `up`):
```
# Generated by NetworkManager
nameserver 10.0.2.3

# Generated by NetworkManager
search lab.local
nameserver 10.0.2.3
nameserver 1.1.1.1
nameserver 8.8.8.8
```

Qué observar: **tras `modify` no cambió nada**; cambió tras `up`. NM junta los DNS de todas las conexiones activas; a igual `ipv4.dns-priority` (100 por defecto) suele poner primero los de la conexión que tiene la ruta por defecto (`enp0s3`), pero el orden exacto puede variar y no importa para el lab. Las consultas a `1.1.1.1` y `8.8.8.8` salen por la ruta por defecto (NAT), no por la tarjeta host-only: el DNS es una propiedad del sistema, no de la interfaz. Esa es la razón para no editar `resolv.conf` a mano: NM lo reescribe en cada activación. Si en un servidor se quiere un DNS fijo aunque el NAT/DHCP entregue otro, se usa `ipv4.ignore-auto-dns yes` en el perfil DHCP.

2. Dos direcciones IP en la misma tarjeta (alias, típico para publicar dos servicios en IPs distintas):

```bash
sudo nmcli connection modify lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"
sudo nmcli connection up lab
ip -br a show enp0s8
sudo grep address /etc/NetworkManager/system-connections/lab.nmconnection
```

Salida esperada:
```
enp0s8           UP             192.168.56.10/24 192.168.56.11/24 fe80::.../64
address1=192.168.56.10/24
address2=192.168.56.11/24
```

Alternativa incremental: `sudo nmcli con mod lab +ipv4.addresses 192.168.56.12/24` añade sin reescribir la lista; `-ipv4.addresses 192.168.56.12/24` la quita. Desde el host se puede hacer `ping 192.168.56.11` para comprobar.

3. Ruta estática persistente (a una red que "está detrás" de otro router) y ruta temporal:

```bash
sudo nmcli connection modify lab +ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route
sudo ip route add 10.20.0.0/24 via 192.168.56.1 dev enp0s8
ip route | grep 10.20
sudo ip route del 10.20.0.0/24
```

Salida esperada (tras el primer `ip route`):
```
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.10.0.0/24 via 192.168.56.1 dev enp0s8 proto static metric 101
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.10 metric 101
```

Qué observar: la ruta de `nmcli` dice `proto static` y sobrevive reinicios (queda en el keyfile como `route1=10.10.0.0/24,192.168.56.1`). La de `ip route add` es inmediata pero desaparece al reiniciar o al hacer `nmcli con up`: sirve para probar antes de fijar. En clase el gateway `192.168.56.1` (el propio host) no reenvía nada; solo se practica la sintaxis. Para dejar el perfil limpio (la ruta apunta a un gateway que solo existe en esta red y estorbaría en el reto), se quita con el prefijo `-`:

```bash
sudo nmcli connection modify lab -ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route | grep 10.10 || echo "ruta 10.10.0.0/24 eliminada"
```

Salida esperada:
```
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)
ruta 10.10.0.0/24 eliminada
```

4. Hostname con FQDN y nombres locales en `/etc/hosts`:

```bash
sudo hostnamectl set-hostname rhel01.lab.local
hostnamectl --static
hostname -s
echo "192.168.56.10 rhel01.lab.local rhel01" | sudo tee -a /etc/hosts
echo "192.168.56.10 servidor-nfs servidor-smb" | sudo tee -a /etc/hosts
getent hosts servidor-nfs
ping -c 1 rhel01.lab.local
```

Salida esperada:
```
rhel01.lab.local
rhel01
192.168.56.10 rhel01.lab.local rhel01
192.168.56.10 servidor-nfs servidor-smb
192.168.56.10  servidor-nfs servidor-smb
PING rhel01.lab.local (192.168.56.10) 56(84) bytes of data.
64 bytes from rhel01.lab.local (192.168.56.10): icmp_seq=1 ttl=64 time=0.045 ms
```

Qué observar: el prompt seguirá diciendo `rhel01`, también en las sesiones nuevas, porque `\h` en `PS1` muestra solo la parte antes del primer punto; el cambio se comprueba con `hostname` y `hostnamectl`. Los alias `servidor-nfs` y `servidor-smb` apuntan a la propia VM: así los labs cliente-servidor de los próximos días (HTTP, NFS, SMB) pueden usar nombres, como en producción, sin necesitar un DNS. El RHCSA formula esto como "configure hostname resolution".

5. MTU y `never-default` (evitar que este perfil tome jamás la ruta por defecto):

```bash
sudo nmcli connection modify lab 802-3-ethernet.mtu 1400 ipv4.never-default yes
sudo nmcli device reapply enp0s8
ip link show enp0s8 | grep mtu
sudo nmcli connection modify lab 802-3-ethernet.mtu 1500
sudo nmcli device reapply enp0s8
```

Salida esperada:
```
Connection successfully reapplied to device 'enp0s8'.
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
```

Qué observar: `device reapply` aplica los cambios del perfil sin bajar la interfaz (no corta sesiones). Si un cambio no se puede aplicar en caliente, devuelve error y hay que usar `con up`. El MTU se toca en la vida real con VPNs o túneles en los que los paquetes grandes se pierden.

6. IPv6: deshabilitar o fijar una dirección estática (objetivo RHCSA):

```bash
sudo nmcli connection modify lab ipv6.method disabled
sudo nmcli connection up lab
ip -br a show enp0s8
sudo nmcli connection modify lab ipv6.method manual ipv6.addresses fd00:56::10/64
sudo nmcli connection up lab
ip -6 addr show enp0s8
```

Salida esperada:
```
enp0s8           UP             192.168.56.10/24 192.168.56.11/24
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 fd00:56::10/64 scope global noprefixroute
    inet6 fe80::5c8e:2b1d:7a40:9c3f/64 scope link noprefixroute
```

Qué observar: con `disabled` desaparece incluso la `fe80::`. `fc00::/7` es el rango de direcciones privadas de IPv6 (ULA, *unique local address*, el equivalente a `192.168.x.x`); en la práctica solo se usa su mitad `fd00::/8`, que es la que uno mismo puede asignarse. Dejar la VM con IPv6 en `manual` o volver a `auto` (`sudo nmcli con mod lab ipv6.method auto ipv6.addresses ""`); cualquiera de los dos es válido para el resto del curso.

7. Editar el archivo a mano y recargar, y la alternativa visual:

```bash
sudo nmcli connection reload
sudo nmcli connection show lab | grep -E '^(ipv4|ipv6|802-3-ethernet)\.(method|addresses|dns|routes|mtu|never-default):'
rpm -q NetworkManager-tui || sudo dnf install -y NetworkManager-tui
sudo nmtui
```

Salida esperada (segunda orden):
```
802-3-ethernet.mtu:                     1500
ipv4.method:                            manual
ipv4.dns:                               1.1.1.1,8.8.8.8
ipv4.addresses:                         192.168.56.10/24, 192.168.56.11/24
ipv4.routes:                            --
ipv4.never-default:                     yes
ipv6.method:                            manual
ipv6.addresses:                         fd00:56::10/64
ipv6.dns:                               --
ipv6.routes:                            --
ipv6.never-default:                     no
```

(Si la ruta `10.10.0.0/24` no se quitó en el paso 3, `ipv4.routes` mostrará `{ ip = 10.10.0.0/24, nh = 192.168.56.1 }`.)

Qué observar en `nmtui`: *Edit a connection* → `lab` → ver los mismos campos (Addresses, Gateway, DNS servers, Search domains, Routing). Se sale con `<Back>` y *Quit* sin cambiar nada. `nmcli con reload` es obligatorio si se editó el `.nmconnection` con un editor: NM no vigila el directorio.

Mención (no práctica): NM también configura **bonding** (`nmcli con add type bond ...` con esclavos, para redundancia de tarjetas), **VLAN** (`nmcli con add type vlan ifname enp0s8.100 dev enp0s8 id 100`) y **bridges**. Son contenido RHCE; en RHEL 9 `teamd` está obsoleto y se recomienda bond.

- **Checkpoint:** pegar en el chat la salida de `ip -br a show enp0s8; getent hosts servidor-nfs; hostnamectl --static`.

---

## Bloque 4 — SSH: acceso remoto y transferencia de archivos

### Conceptos (8 min)

SSH es la herramienta con la que se administra el 100 % de los servidores Linux. Tiene dos partes: el **servidor** `sshd` (servicio systemd, puerto 22, ya activo desde la instalación) y el **cliente** `ssh` (en la VM y en el equipo del participante, incluido Windows 10/11 con OpenSSH en PowerShell).

**Configuración del servidor en RHEL 9.** `/etc/ssh/sshd_config` empieza con `Include /etc/ssh/sshd_config.d/*.conf`: lo que esté en ese directorio **gana** sobre el archivo principal porque se lee primero y en OpenSSH la primera aparición de una directiva es la que vale. Red Hat deja ahí `50-redhat.conf` (integración con crypto-policies, PAM, logging) y el instalador crea `01-permitrootlogin.conf` si se marcó "permitir root por SSH con contraseña". Regla práctica: para cambiar algo se crea un archivo propio en `sshd_config.d/` con número bajo, se valida con `sshd -t` y se ve el resultado efectivo con `sshd -T`. El hardening (prohibir root, prohibir contraseñas) se hace el Día 8; hoy solo se mira.

**Autenticación por clave.** Un par de claves: la **privada** (`~/.ssh/id_ed25519`, nunca sale del equipo, permisos `600`) y la **pública** (`.pub`, se copia al servidor en `~/.ssh/authorized_keys`). `ed25519` es el algoritmo recomendado hoy: corto, rápido y seguro; `rsa` de 4096 bits sigue siendo válido para sistemas antiguos. `sshd` es estricto con permisos (`StrictModes yes`): ni el home, ni `~/.ssh`, ni `authorized_keys` pueden ser escribibles por grupo u otros; lo habitual y recomendado es `~/.ssh` en `700` y `authorized_keys` en `600`. Si no se cumple, ignora la clave sin avisar al cliente (el motivo solo aparece en el log del servidor).

**Huella del servidor.** La primera vez que se entra, `ssh` guarda la huella del servidor en `~/.ssh/known_hosts`. Si cambia (se reinstaló el servidor, o se restauró un snapshot con claves distintas, o alguien está en medio), aparece `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` y se rechaza la conexión. La solución correcta es verificar la huella y borrar la entrada vieja con `ssh-keygen -R host`, nunca deshabilitar la comprobación.

### Lab 4.1 — Claves SSH: desde el equipo propio y dentro de la VM (15 min)

- **Objetivo:** entrar a la VM sin contraseña desde el equipo propio, y practicar `ssh-copy-id` dentro de la VM contra `localhost`.

**Parte A — en el equipo del participante** (PowerShell en Windows, Terminal en macOS/Linux).

1. Generar el par de claves. Se puede dejar la passphrase vacía para el curso (en producción se pone y se usa `ssh-agent`):

```bash
ssh-keygen -t ed25519 -C "curso-rhel"
```

Salida esperada:
```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/ana/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Your identification has been saved in /Users/ana/.ssh/id_ed25519
Your public key has been saved in /Users/ana/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:Qm4k... curso-rhel
```

Qué observar: en Windows la ruta es `C:\Users\Ana\.ssh\id_ed25519`. Si el archivo ya existe (por ejemplo del Día 1), responder `n` a sobrescribir y usar el existente.

2. Copiar la clave pública a la VM.

macOS / Linux:
```bash
ssh-copy-id -p 2222 student@localhost
```

Windows PowerShell (no existe `ssh-copy-id`; se hace lo mismo a mano):
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 student@localhost "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Salida esperada (macOS/Linux):
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/ana/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
student@localhost's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh -p '2222' 'student@localhost'"
```

3. Probar y, si algo falla, depurar con `-v`:

```bash
ssh -p 2222 student@localhost hostname
ssh -v -p 2222 student@localhost exit 2>&1 | grep -E "Offering public key|Server accepts key|Authenticated"
```

Salida esperada:
```
rhel01.lab.local
debug1: Offering public key: /Users/ana/.ssh/id_ed25519 ED25519 SHA256:Qm4k... curso-rhel
debug1: Server accepts key: /Users/ana/.ssh/id_ed25519 ED25519 SHA256:Qm4k... curso-rhel
Authenticated to localhost ([127.0.0.1]:2222) using "publickey".
```

Qué observar: entró sin pedir contraseña. La forma `ssh host comando` ejecuta y sale: es la base de la automatización remota (Día 9 la usa en scripts). `-v` muestra qué clave se ofrece y si el servidor la acepta; con `-vvv` se ve todo el intercambio.

**Parte B — dentro de la VM**, para que todos (incluidos los de Windows) practiquen `ssh-copy-id` y vean los permisos:

4. Generar un par para `student` en la VM y copiarlo al propio servidor:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "student@rhel01"
ssh-copy-id student@localhost
ssh localhost hostname
```

Salida esperada (resumida):
```
Your public key has been saved in /home/student/.ssh/id_ed25519.pub
The authenticity of host 'localhost (::1)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
student@localhost's password:
Number of key(s) added: 1
rhel01.lab.local
```

5. Revisar permisos y contenido, y comprobar la huella del servidor para compararla con la que mostró el cliente:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh
cat ~/.ssh/authorized_keys
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Salida esperada:
```
drwx------. 2 student student  98 sep  3 10:12 /home/student/.ssh
-rw-------. 1 student student 188 sep  3 10:12 authorized_keys
-rw-------. 1 student student 411 sep  3 10:10 id_ed25519
-rw-r--r--. 1 student student  96 sep  3 10:10 id_ed25519.pub
-rw-r--r--. 1 student student 173 sep  3 10:11 known_hosts
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... curso-rhel
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... student@rhel01
256 SHA256:oR4x... no comment (ED25519)
```

(La clave de host la genera `sshd-keygen` sin comentario, por eso dice `no comment`.)

Qué observar: dos claves autorizadas (la del equipo propio y la de la propia VM). El punto tras los permisos (`drwx------.`) indica contexto SELinux (Día 8). Si alguien copió la clave con `cat >>` sin `chmod`, aquí se ve y se corrige: `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys`.

6. *(Si sobra tiempo; si no, demo del instructor.)* Agente de claves (para claves con passphrase; se muestra el flujo aunque la clave no la tenga):

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
ssh-add -l
```

Salida esperada:
```
Agent pid 2431
Identity added: /home/student/.ssh/id_ed25519 (student@rhel01)
256 SHA256:... student@rhel01 (ED25519)
```

Qué observar: el agente guarda la clave descifrada en memoria durante la sesión; se escribe la passphrase una vez. En Windows, el servicio `ssh-agent` viene deshabilitado: se activa en PowerShell como administrador con `Set-Service ssh-agent -StartupType Manual; Start-Service ssh-agent` y luego `ssh-add`.

- **Checkpoint:** pegar en el chat la salida de `ls -ld ~/.ssh; wc -l ~/.ssh/authorized_keys` (esperado: `drwx------` y 2 líneas).

### Lab 4.2 — ~/.ssh/config y known_hosts (8 min)

- **Objetivo:** crear alias de conexión en el equipo propio y saber reaccionar cuando cambia la huella de un servidor.

1. En el **equipo propio**, crear `~/.ssh/config` con dos alias: uno por la IP host-only y otro por el port forwarding.

macOS / Linux:
```bash
cat >> ~/.ssh/config <<'EOF'
Host rhel01
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519

Host rhel01-nat
    HostName localhost
    Port 2222
    User student
    IdentityFile ~/.ssh/id_ed25519
EOF
chmod 600 ~/.ssh/config
```

Windows PowerShell (el archivo se llama `config`, sin extensión):
```powershell
New-Item -ItemType File -Force $env:USERPROFILE\.ssh\config | Out-Null
notepad $env:USERPROFILE\.ssh\config
```

y pegar el mismo contenido (en UTM/instructor, `HostName` es la IP del rango host-only del Mac).

2. Usar los alias:

```bash
ssh rhel01 hostname
ssh rhel01-nat "ip -br a show enp0s8"
```

Salida esperada:
```
rhel01.lab.local
enp0s8           UP             192.168.56.10/24 192.168.56.11/24 fd00:56::10/64 fe80::.../64
```

Qué observar: el alias también funciona con `scp`, `sftp` y `rsync` (`scp archivo rhel01:/tmp/`). En un puesto de administración real, `~/.ssh/config` tiene decenas de servidores con sus puertos, usuarios y claves.

3. Simular un cambio de huella. Desde el equipo propio, borrar la entrada guardada y volver a conectar:

```bash
ssh-keygen -R 192.168.56.10
ssh-keygen -R "[localhost]:2222"
ssh rhel01 hostname
```

Salida esperada:
```
# Host 192.168.56.10 found: line 3
/Users/ana/.ssh/known_hosts updated.
Original contents retained as /Users/ana/.ssh/known_hosts.old
# Host [localhost]:2222 found: line 1
/Users/ana/.ssh/known_hosts updated.
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:oR4x...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
rhel01.lab.local
```

Qué observar: la huella `SHA256:oR4x...` debe coincidir con la que dio `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` en la VM. Eso es "verificar la huella". Cuando al restaurar un snapshot o reinstalar aparezca `REMOTE HOST IDENTIFICATION HAS CHANGED`, el procedimiento es exactamente este: `ssh-keygen -R host` y comprobar. Aviso: como también se borró la entrada `[localhost]:2222`, la siguiente conexión por `rhel01-nat` volverá a preguntar por la huella; se responde `yes` una vez y queda registrada de nuevo.

- **Checkpoint:** pegar en el chat la salida de `ssh rhel01 hostname` (o `ssh rhel01-nat hostname`).

### Lab 4.3 — scp, sftp, rsync y un túnel local (14 min)

- **Objetivo:** mover archivos en ambas direcciones y llegar a un servicio interno de la VM a través de SSH.

1. `scp` desde el equipo propio a la VM y de vuelta (con los alias del lab anterior):

```bash
echo "nota desde mi equipo $(date)" > nota.txt
scp nota.txt rhel01:empresa/documentos/
scp rhel01:/etc/hostname ./hostname-rhel01.txt
cat hostname-rhel01.txt
```

Salida esperada:
```
nota.txt                                      100%   45    22.1KB/s   00:00
hostname                                      100%   17     8.3KB/s   00:00
rhel01.lab.local
```

Qué observar: sintaxis `origen destino`; el lado remoto es `host:ruta`. Una ruta remota **sin** `/` inicial es relativa al home del usuario remoto (`empresa/documentos/` = `/home/student/empresa/documentos/`): es la forma más portable, porque la expansión de `~` en el lado remoto depende de la versión de OpenSSH del cliente. Sin alias sería `scp -P 2222 nota.txt student@localhost:~/empresa/documentos/` (`-P` mayúscula en `scp` y `sftp`, `-p` minúscula en `ssh`: error clásico). En Windows, `scp` está incluido con OpenSSH y funciona igual (`scp nota.txt rhel01:/home/student/empresa/documentos/`); el archivo de prueba se crea en PowerShell con `"nota desde mi equipo $(Get-Date)" | Out-File -Encoding ascii nota.txt` (el `>` de PowerShell escribiría UTF-16) y `cat` se usa igual.

2. *(Si sobra tiempo.)* `sftp` interactivo (útil cuando no se sabe la ruta exacta):

```bash
sftp rhel01
```

Dentro de la sesión:
```
sftp> pwd
Remote working directory: /home/student
sftp> ls empresa/documentos
empresa/documentos/doc1.txt  ...  empresa/documentos/nota.txt
sftp> get empresa/documentos/nota.txt nota-copia.txt
sftp> lls
sftp> put hostname-rhel01.txt /tmp/
sftp> bye
```

Qué observar: comandos con `l` delante (`lls`, `lcd`, `lpwd`) actúan en el lado local. `get` descarga, `put` sube.

3. `rsync` **dentro de la VM**: copia incremental, primero local y luego por SSH contra `localhost`:

```bash
sudo dnf install -y rsync
rsync -avz --delete ~/empresa/ ~/empresa-copia/
touch ~/empresa/documentos/nuevo.txt
rm ~/empresa/documentos/doc1.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
rsync -avz --delete ~/empresa/ student@localhost:/tmp/empresa-remota/
ls /tmp/empresa-remota/documentos/
touch ~/empresa/documentos/doc1.txt
rm -f ~/empresa/documentos/nuevo.txt
```

Salida esperada (segunda ejecución local):
```
sending incremental file list
deleting documentos/doc1.txt
documentos/
documentos/nuevo.txt

sent 312 bytes  received 58 bytes  740.00 bytes/sec
total size is 45  speedup is 0.12
```

Qué observar: la segunda vez solo viaja lo que cambió (`nuevo.txt`) y `--delete` borra en destino lo que ya no está en origen (`doc1.txt`): así se mantiene un espejo. **La barra final importa**: `~/empresa/` copia el contenido; `~/empresa` (sin barra) crearía `empresa-copia/empresa/`. `-a` conserva permisos, dueños y fechas; `-v` verboso; `-z` comprime en tránsito. `rsync` por SSH es la base de `backup.sh` hacia otro servidor (Día 9). Las dos últimas líneas devuelven `~/empresa/documentos/` al estado del Día 2 (`doc1.txt` a `doc5.txt`), que es el que dan por supuesto los días siguientes.

4. *(Si falta tiempo, demo del instructor; los participantes lo repiten en la tarea.)* Túnel local: llegar a un puerto de la VM que **no** está publicado ni por port forwarding ni por firewall. En la VM, levantar un servidor web de prueba con Python (`python3` viene en la instalación base). Se usa el 8000 y no el 80 del `httpd` instalado el Día 4 justamente porque el 80 **sí** está publicado por el port forwarding `8080→80` y no serviría para demostrar el túnel:

```bash
cd ~/empresa && python3 -m http.server 8000
```

Salida esperada:
```
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Desde el **equipo propio**, en otra terminal, primero comprobar que directo no llega y luego abrir el túnel:

```bash
curl -m 3 http://192.168.56.10:8000/ ; echo "codigo: $?"
ssh -N -L 8081:localhost:8000 rhel01
```

(En PowerShell escribir `curl.exe`, porque `curl` a secas es un alias de `Invoke-WebRequest`.)

En el navegador del equipo propio: `http://localhost:8081/`.

Salida esperada (macOS/Linux; firewalld rechaza con ICMP "admin prohibited", por eso falla al instante y no por timeout):
```
curl: (7) Failed to connect to 192.168.56.10 port 8000 after 2 ms: No route to host
codigo: 7
```
(En Windows el mensaje puede ser `Connection refused` o un timeout con código 28; lo importante es que **no** devuelve el listado.) En el navegador, el listado de `~/empresa` (`documentos/`, `clientes/`, `backups/`, `logs/`). En la terminal de la VM aparecen las peticiones: `127.0.0.1 - - [03/Sep/2026 10:41:02] "GET / HTTP/1.1" 200 -`.

Qué observar: directo a `:8000` no llega porque **firewalld** solo permite `ssh`, `cockpit` y `dhcpv6-client` en la zona `public` (se abre el Día 8). El túnel entra por el 22, que sí está permitido, y desde dentro de la VM habla con `localhost:8000`, donde el firewall no interviene. `-L puerto_local:destino_visto_desde_el_servidor:puerto`; `-N` = no abrir shell. Es el truco de administración para consolas web internas (Cockpit en el 9090, bases de datos) sin abrir puertos. Cerrar el túnel con `Ctrl+C` y detener `python3` con `Ctrl+C` en la VM.

- **Checkpoint:** pegar en el chat la última línea de `rsync -avz --delete ~/empresa/ ~/empresa-copia/` (`total size is ... speedup is ...`) y el `GET / HTTP/1.1" 200` del servidor de prueba si llegaron al túnel.

---

## Bloque 5 — Troubleshooting de red: método por capas

### Conceptos (8 min)

Un ticket de red nunca dice "el gateway está mal"; dice "no funciona el correo" o "no puedo entrar al servidor". El método es recorrer las capas **de abajo hacia arriba**, una pregunta por capa, y detenerse en la primera que falla. Se avanza solo cuando la capa actual responde bien.

| # | Pregunta | Comando | Respuesta sana |
|---|---|---|---|
| 1 | ¿Hay enlace? | `ip link`, `nmcli device status` | `state UP`, `LOWER_UP`, `connected` |
| 2 | ¿Tengo la IP correcta? | `ip -br a`, `nmcli con show NOMBRE` | La IP y la máscara esperadas, en la interfaz esperada |
| 3 | ¿Llego al gateway? | `ip route`, `ping -c 3 GATEWAY`, `ip neigh` | `default via` correcto, 0 % pérdida, vecino `REACHABLE` |
| 4 | ¿Salgo a internet por IP? | `ping -c 3 8.8.8.8`, `tracepath -n 8.8.8.8` | Respuesta; si falla aquí, el problema es routing/NAT/proveedor |
| 5 | ¿Resuelvo nombres? | `getent hosts redhat.com`, `dig redhat.com`, `dig @8.8.8.8 redhat.com`, `cat /etc/resolv.conf`, `cat /etc/hosts` | `getent` responde; si solo `@8.8.8.8` responde, el DNS configurado está mal |
| 6 | ¿El servicio escucha? | `sudo ss -tulpn`, `systemctl status SERVICIO` | `LISTEN` en `0.0.0.0:PUERTO` o en la IP correcta |
| 7 | ¿El firewall lo deja pasar? | `sudo firewall-cmd --list-all` | El servicio/puerto en la lista (se profundiza el Día 8) |
| 8 | ¿SELinux lo bloquea? | `getenforce`, `sudo ausearch -m avc -ts recent` | Se profundiza el Día 8 |

Tabla rápida de síntomas:

| Síntoma que reporta el usuario | Capa sospechosa | Primer comando |
|---|---|---|
| "No tengo red, nada funciona" | 1–2 | `nmcli device status; ip -br a` |
| "`ping 8.8.8.8` falla, pero llego a los equipos de mi red" | 3 | `ip route` (¿hay `default via`? ¿es el correcto?) |
| "`ping 8.8.8.8` funciona pero `ping google.com` no" | 5 | `cat /etc/resolv.conf; dig @8.8.8.8 google.com` |
| "Resuelve lento (5–10 s) y luego funciona" | 5 | Primer `nameserver` de `resolv.conf` no responde; `dig` tarda y termina en `;; connection timed out; no servers could be reached` |
| "Desde mi PC no llego al servidor pero él sí sale a internet" | 6–7 | `ss -tulpn` en el servidor; `firewall-cmd --list-all`; `tcpdump` para ver si el tráfico llega |
| "Funciona por IP pero no por nombre corto" | 5 | `/etc/hosts`, `ipv4.dns-search` |
| "Después de reiniciar perdí la IP fija" | 2 | Se configuró con `ip addr add` (volátil) en vez de `nmcli`; `nmcli con show` |
| "Cambié el DNS en resolv.conf y volvió a lo de antes" | 5 | NM lo reescribió; usar `nmcli con mod ... ipv4.dns` |

Dos herramientas más para cuando la tabla no alcanza: `journalctl -u NetworkManager` cuenta qué hizo NM y cuándo (DHCP fallido, cable desconectado, perfil activado), y `tcpdump` responde la pregunta definitiva "¿está llegando el tráfico o no?": si el paquete no aparece en `tcpdump`, el problema está antes del servidor (red, hipervisor, firewall externo); si aparece y no hay respuesta, está en el servidor (servicio, firewall local, SELinux).

### Lab 5.1 — Diagnóstico guiado, journalctl y tcpdump (17 min)

- **Objetivo:** recorrer el método sobre una falla provocada y ver el tráfico con `tcpdump`.

1. Instalar `tcpdump` y leer lo que NM ha hecho hoy:

```bash
sudo dnf install -y tcpdump
journalctl -u NetworkManager -n 15 --no-pager
journalctl -u NetworkManager --since "1 hour ago" | grep -iE "lab|enp0s8" | tail -5
```

Salida esperada (resumida):
```
sep 03 10:20:11 rhel01 NetworkManager[812]: <info>  [1756901211.4021] device (enp0s8): state change: prepare -> config (reason 'none', ...)
sep 03 10:20:11 rhel01 NetworkManager[812]: <info>  [1756901211.4102] device (enp0s8): state change: ip-config -> ip-check ...
sep 03 10:20:11 rhel01 NetworkManager[812]: <info>  [1756901211.4188] device (enp0s8): Activation: successful, device activated.
```

Qué observar: cada activación deja rastro con la hora. Ante un "se cayó la red a las 3 am", este log dice si fue el cable (`carrier: link disconnected`), el DHCP (`dhcp4 (enp0s3): request timed out`) o alguien que ejecutó `con down`.

2. Provocar una falla y recorrer las capas. Bajar la conexión `lab`:

```bash
sudo nmcli connection down lab
nmcli device status
ip -br a
ip route
```

Salida esperada:
```
DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
enp0s8  ethernet  disconnected            --
lo      loopback  connected (externally)  lo
enp0s8           UP             fe80::5c8e:2b1d:7a40:9c3f/64
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
```

Qué observar: la capa 1 muestra el dispositivo `disconnected` sin perfil; en la capa 2 la interfaz sigue `UP` (NM la deja levantada para detectar el cable) pero **sin IPv4** (según la versión de NM puede quedar solo la `fe80::` o nada; ⚠️ Verificar en la VM antes de la clase), y desapareció la ruta a `192.168.56.0/24`. No hace falta seguir bajando: la falla está en la capa 1–2. Reparar:

```bash
sudo nmcli connection up lab
ip -br a show enp0s8
```

3. Mirar la interfaz con todos los detalles que NM conoce (útil cuando la IP "está" pero algo no cuadra):

```bash
nmcli -f all device show enp0s8 | head -25
```

Salida esperada (resumida):
```
GENERAL.DEVICE:                         enp0s8
GENERAL.TYPE:                           ethernet
GENERAL.NM-TYPE:                        NMDeviceEthernet
GENERAL.DBUS-PATH:                      /org/freedesktop/NetworkManager/Devices/3
GENERAL.VENDOR:                         Intel Corporation
GENERAL.PRODUCT:                        82540EM Gigabit Ethernet Controller
GENERAL.DRIVER:                         e1000
GENERAL.HWADDR:                         08:00:27:8E:1A:2B
GENERAL.MTU:                            1500
GENERAL.STATE:                          100 (connected)
GENERAL.REASON:                         0 (No reason given)
GENERAL.IP4-CONNECTIVITY:               3 (limited)
GENERAL.CONNECTION:                     lab
GENERAL.AUTOCONNECT:                    yes
GENERAL.FIRMWARE-MISSING:               no
WIRED-PROPERTIES.CARRIER:               on
```

Qué observar: `DRIVER`, `CARRIER` y `AUTOCONNECT`. `IP4-CONNECTIVITY: limited` en `enp0s8` es normal (tiene IP pero no ruta por defecto: host-only no sale a internet); en `enp0s3` dirá `4 (full)`. Sin el paquete `NetworkManager-config-connectivity-redhat` NM no hace comprobación real y deduce el valor de la tabla de rutas (⚠️ Verificar en la VM antes de la clase el valor exacto).

4. `tcpdump` mientras llega un ping. Abrir una **segunda sesión SSH** a la VM. En la primera:

```bash
sudo tcpdump -i enp0s8 -n icmp
```

Desde el equipo propio (o desde la segunda sesión hacia la propia IP; en Windows `ping -n 4 192.168.56.10`, que envía 32 bytes y se verá como `length 40`):

```bash
ping -c 4 192.168.56.10
```

Salida esperada en la primera sesión:
```
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10:52:01.118273 IP 192.168.56.1 > 192.168.56.10: ICMP echo request, id 18, seq 1, length 64
10:52:01.118305 IP 192.168.56.10 > 192.168.56.1: ICMP echo reply, id 18, seq 1, length 64
10:52:02.120114 IP 192.168.56.1 > 192.168.56.10: ICMP echo request, id 18, seq 2, length 64
10:52:02.120141 IP 192.168.56.10 > 192.168.56.1: ICMP echo reply, id 18, seq 2, length 64
```

Detener con `Ctrl+C`. Qué observar: se ve la pregunta (`request`) y la respuesta (`reply`). Si se viera solo `request` sin `reply`, el servidor recibe pero no contesta (firewall local con ICMP bloqueado, por ejemplo). Si no se viera nada, el paquete nunca llegó.

5. Ver el tráfico SSH y el DNS. En la primera sesión:

```bash
sudo tcpdump -i enp0s3 -n port 22 -c 6
```

Después, en la primera sesión dejar capturando DNS (queda esperando):

```bash
sudo tcpdump -i any -n udp port 53 -c 2
```

y en la segunda sesión provocar una consulta:

```bash
dig +short redhat.com
```

Salida esperada (primera sesión; el servidor destino es el **primer** `nameserver` de `/etc/resolv.conf`, normalmente `10.0.2.3`; si en la VM aparece `1.1.1.1` primero, se verá esa IP):
```
10:55:10.301122 enp0s3 Out IP 10.0.2.15.44120 > 10.0.2.3.53: 51234+ [1au] A? redhat.com. (51)
10:55:10.340871 enp0s3 In  IP 10.0.2.3.53 > 10.0.2.15.44120: 51234 1/0/1 A 34.235.198.240 (55)
```

Qué observar: `-c N` detiene tras N paquetes (imprescindible al capturar el puerto 22 mientras se está conectado por SSH: la propia salida de `tcpdump` genera tráfico SSH y se retroalimenta). No se lanza `sudo tcpdump ... &` en segundo plano: si `sudo` pide contraseña, el proceso queda detenido esperando la terminal. En la captura DNS se ve la pregunta `A? redhat.com.` hacia el `nameserver` de `resolv.conf` y la respuesta. Para guardar una captura y analizarla en Wireshark: `sudo tcpdump -i enp0s3 -w /tmp/captura.pcap -c 200` y luego `scp`.

- **Checkpoint:** pegar en el chat dos líneas de la captura ICMP (un `request` y su `reply`).

---

## Reto individual (20 min)

**Ticket #RED-05.** "Desde esta mañana el servidor `rhel01` tiene problemas de red. Algunos usuarios dicen que no pueden entrar por la IP del laboratorio; otros, que el servidor no actualiza paquetes ni resuelve nombres. No sabemos qué se tocó. Déjelo como estaba y documente qué encontró."

El instructor comparte el script `romper-red.sh`. El participante lo ejecuta en su VM **desde la consola del hipervisor** (ventana de VirtualBox/UTM) o, si prefiere seguir por SSH, con la opción `--solo-lab`, que no toca el perfil NAT:

```bash
sudo bash romper-red.sh            # modo completo: puede tocar NAT y lab (usar la consola de la VM)
sudo bash romper-red.sh --solo-lab # modo seguro: solo el perfil lab
```

El script aplica **una o dos fallas al azar** entre: gateway inválido en el perfil NAT, DNS inexistente, conexión `lab` desactivada, IP de `lab` en otra red. El participante debe:

1. Recorrer las capas del Bloque 5 en orden y anotar, por cada falla, **qué comando la delató** y qué salida vio.
2. Corregir con `nmcli` (no con `ip addr`/`ip route`, porque no sería persistente).
3. Verificar: `ping -c 2 8.8.8.8`, `getent hosts redhat.com`, y desde el equipo propio `ssh rhel01 hostname`.
4. Entregar en el chat una tabla `Falla | Comando que la delató | Comando de corrección`.

Sin pistas adicionales. El registro `/root/romper-red.log` es solo para el instructor: no consultarlo hasta terminar.

### Script `romper-red.sh` (para el instructor)

```bash
#!/bin/bash
# romper-red.sh — Día 05. Introduce 1 o 2 fallas de red al azar. USO DEL INSTRUCTOR.
#   sudo bash romper-red.sh             -> puede tocar el perfil NAT (gateway/DNS) y el perfil lab
#   sudo bash romper-red.sh --solo-lab  -> solo toca el perfil lab (seguro si se trabaja por SSH NAT)
# Deja respaldo de los perfiles en /root/nm-backup-FECHA y registro en /root/romper-red.log
set -u
[[ $EUID -eq 0 ]] || { echo "Ejecutar con sudo"; exit 1; }

LAB_CON="lab"
NAT_IF=$(ip -4 route show default | awk '{print $5; exit}')
NAT_CON=$(nmcli -g GENERAL.CONNECTION device show "$NAT_IF")
LOG=/root/romper-red.log
BACKUP=/root/nm-backup-$(date +%F-%H%M%S)

if [[ -z "$NAT_IF" || -z "$NAT_CON" ]]; then
  echo "No se pudo detectar la interfaz/perfil de la ruta por defecto. Revisar 'ip route'."; exit 1
fi

if ! nmcli -g NAME connection show | grep -qx "$LAB_CON"; then
  echo "No existe el perfil '$LAB_CON'. Completar el Lab 3.2 primero."; exit 1
fi

mkdir -p "$BACKUP"
cp /etc/NetworkManager/system-connections/*.nmconnection "$BACKUP"/
{
  echo "==== $(date '+%F %T') ===="
  echo "Respaldo de perfiles: $BACKUP"
  echo "Perfil NAT: $NAT_CON ($NAT_IF)   Perfil lab: $LAB_CON"
} >> "$LOG"

falla_gateway() {
  local addr dns
  addr=$(ip -4 -o addr show dev "$NAT_IF" | awk '{print $4; exit}')
  dns=$(nmcli -g IP4.DNS device show "$NAT_IF" | tr '|' '\n' | grep -v '^$' | paste -sd, -)
  [[ -n "$dns" ]] || dns="10.0.2.3"
  nmcli connection modify "$NAT_CON" ipv4.method manual ipv4.addresses "$addr" \
      ipv4.gateway 10.0.2.254 ipv4.dns "$dns"
  nmcli connection up "$NAT_CON" >/dev/null
  echo "FALLA gateway : $NAT_CON pasado a manual ($addr) con gateway 10.0.2.254 (inexistente)" >> "$LOG"
}

falla_dns() {
  nmcli connection modify "$NAT_CON" ipv4.ignore-auto-dns yes ipv4.dns 192.0.2.53
  nmcli connection modify "$LAB_CON" ipv4.dns 192.0.2.53
  nmcli connection up "$NAT_CON" >/dev/null
  nmcli connection up "$LAB_CON" >/dev/null 2>&1
  echo "FALLA dns     : DNS 192.0.2.53 (inexistente) en $NAT_CON (ignore-auto-dns yes) y en $LAB_CON" >> "$LOG"
}

falla_lab_ip() {
  local orig
  orig=$(nmcli -g ipv4.addresses connection show "$LAB_CON")
  nmcli connection modify "$LAB_CON" ipv4.addresses 192.168.57.10/24
  nmcli connection up "$LAB_CON" >/dev/null 2>&1
  echo "FALLA lab_ip  : $LAB_CON cambiado de '$orig' a 192.168.57.10/24 (red equivocada)" >> "$LOG"
}

falla_lab_down() {
  nmcli connection modify "$LAB_CON" connection.autoconnect no
  nmcli connection down "$LAB_CON" >/dev/null 2>&1
  echo "FALLA lab_down: $LAB_CON desactivado y connection.autoconnect=no" >> "$LOG"
}

if [[ "${1:-}" == "--solo-lab" ]]; then
  CANDIDATAS=(lab_ip lab_down)
else
  CANDIDATAS=(gateway dns lab_ip lab_down)
fi
N=$(shuf -i 1-2 -n 1)
ELEGIDAS=$(shuf -e "${CANDIDATAS[@]}" -n "$N")

# Orden fijo: lab_down siempre al final para que ninguna otra falla lo reactive
for f in gateway dns lab_ip lab_down; do
  grep -qx "$f" <<< "$ELEGIDAS" && "falla_$f"
done

echo "Listo: $N falla(s) aplicada(s). A diagnosticar por capas."
echo "Registro (solo instructor): $LOG"
```

Nota de seguridad del script: la falla `gateway` conserva la IP NAT actual (`10.0.2.15`), por lo que el port forwarding del hipervisor normalmente sigue funcionando (el hipervisor habla con la VM en la misma red, sin pasar por el gateway; ⚠️ Verificar en la VM antes de la clase, en VirtualBox y en UTM). La falla `lab_ip` supone que el perfil `lab` ya no tiene la ruta estática `10.10.0.0/24 via 192.168.56.1` (se quitó al final del paso 3 del Lab 3.3): si quedara, al cambiar la IP a otra red el gateway de esa ruta deja de ser alcanzable y `nmcli con up lab` puede fallar o dejar un aviso en `journalctl -u NetworkManager`. Aun así, reactivar el perfil NAT puede congelar unos segundos la sesión SSH, y en un servidor real un cambio así **sí** dejaría fuera al administrador: por eso se recomienda la consola. El script se prueba antes de clase en la VM del instructor, en ambos modos.

### Solución (para el instructor)

Diagnóstico esperado por capas y corrección:

| Falla | Síntoma | Comando que la delata | Corrección |
|---|---|---|---|
| `gateway` | `ping 8.8.8.8` → `From 10.0.2.15 icmp_seq=1 Destination Host Unreachable` (⚠️ Verificar en la VM: el NAT del hipervisor no debería responder ARP por `10.0.2.254`); `ping 10.0.2.2` funciona; `dig` sigue funcionando si `10.0.2.3` quedó como DNS (está en la misma red, no necesita gateway), aunque tarda varios segundos si `1.1.1.1`/`8.8.8.8` van antes en `resolv.conf` (esos sí necesitan gateway y agotan su timeout); `dnf` no descarga | `ip route` muestra `default via 10.0.2.254`; `nmcli con show enp0s3 \| grep -E 'ipv4.(method\|gateway)'` → `manual` y `10.0.2.254`; `ip neigh` → `10.0.2.254 ... FAILED` | `sudo nmcli con mod enp0s3 ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""` y `sudo nmcli con up enp0s3` |
| `dns` | `ping 8.8.8.8` funciona; `ping redhat.com` → `Temporary failure in name resolution` (tras varios segundos); `dig redhat.com` → `;; connection timed out; no servers could be reached` (tarda ~10 s); `dig @8.8.8.8 redhat.com` funciona | `cat /etc/resolv.conf` → `nameserver 192.0.2.53`; `nmcli con show enp0s3 \| grep -E 'ipv4.(dns\|ignore-auto-dns)'`; `nmcli con show lab \| grep ipv4.dns` | `sudo nmcli con mod enp0s3 ipv4.ignore-auto-dns no ipv4.dns ""`; `sudo nmcli con mod lab ipv4.dns "1.1.1.1 8.8.8.8"`; `sudo nmcli con up enp0s3; sudo nmcli con up lab` |
| `lab_ip` | Desde el host, `ssh rhel01` → `Connection timed out` / `No route to host`; internet funciona | `ip -br a` → `enp0s8 ... 192.168.57.10/24`; `nmcli -g ipv4.addresses con show lab` | `sudo nmcli con mod lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"` y `sudo nmcli con up lab` (en UTM, la IP del rango host-only del Mac) |
| `lab_down` | `ssh rhel01` desde el host falla; `ip -br a` → `enp0s8` sin IPv4 (solo `fe80::` o nada) | `nmcli device status` → `enp0s8 disconnected --`; `nmcli con show` → `lab` sin `DEVICE`; `nmcli -g connection.autoconnect con show lab` → `no` | `sudo nmcli con mod lab connection.autoconnect yes` y `sudo nmcli con up lab` |

Verificación final que se exige: `ip route` con `default via 10.0.2.2`, `getent hosts redhat.com` responde, `cat /etc/resolv.conf` sin `192.0.2.53`, `ip -br a show enp0s8` con la IP correcta, y `ssh rhel01 hostname` desde el equipo propio.

Restauración de emergencia (si un participante se enreda y el tiempo apremia), usando el respaldo que dejó el script:

```bash
sudo -i
ls -d /root/nm-backup-*
cp /root/nm-backup-*/*.nmconnection /etc/NetworkManager/system-connections/
nmcli connection reload
nmcli connection up enp0s3
nmcli connection up lab
cat /root/romper-red.log
exit
```

Puntos de evaluación: (1) recorrió las capas en orden y se detuvo en la correcta, (2) corrigió con `nmcli` y no con comandos volátiles, (3) verificó desde fuera de la VM, (4) la tabla entregada nombra el comando que delató cada falla.

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `ip -br a` / `ip addr` | Interfaces y direcciones IP (breve / completo) |
| `ip link` / `ip -s link show IF` | Estado del enlace (UP, LOWER_UP, MTU) y contadores de errores |
| `ip route` | Tabla de rutas; `default via` es el gateway |
| `sudo ip route add RED via GW dev IF` | Ruta temporal (se pierde al reiniciar) |
| `ip neigh` | Tabla ARP: vecinos y sus MAC |
| `ping -c 3 DESTINO` | Conectividad ICMP |
| `tracepath -n IP` / `traceroute -n IP` / `mtr -n IP` | Camino salto a salto |
| `sudo ss -tulpn` | Puertos TCP/UDP en escucha y su proceso |
| `ss -tan` | Todas las conexiones TCP (numérico) |
| `hostnamectl set-hostname FQDN` | Fijar el hostname (persistente) |
| `getent hosts NOMBRE` | Resolver como lo hace el sistema (hosts → DNS) |
| `dig NOMBRE` / `dig +short` / `dig @SERVIDOR NOMBRE` / `dig -x IP` | Consultas DNS (completa, corta, a un servidor concreto, inversa) |
| `nmcli device status` | Dispositivos y qué perfil tienen activo |
| `nmcli connection show [NOMBRE]` | Perfiles; con nombre, todas sus propiedades |
| `nmcli device show IF` | Datos en vivo de una interfaz (IP, gateway, DNS, rutas) |
| `sudo nmcli con add type ethernet con-name N ifname IF ipv4.method manual ipv4.addresses IP/PFX` | Crear perfil con IP estática |
| `sudo nmcli con mod N ipv4.dns X +ipv4.dns Y ipv4.dns-search DOM` | Fijar DNS y dominio de búsqueda |
| `sudo nmcli con mod N ipv4.addresses "A/24,B/24"` | Varias IPs en la interfaz |
| `sudo nmcli con mod N +ipv4.routes "RED GW"` | Ruta estática persistente |
| `sudo nmcli con mod N ipv6.method disabled` | Apagar IPv6 en un perfil |
| `sudo nmcli con up N` / `down N` | Aplicar / desactivar un perfil |
| `sudo nmcli device reapply IF` | Aplicar cambios sin bajar la interfaz |
| `sudo nmcli con reload` | Releer keyfiles editados a mano |
| `sudo nmcli con delete N` | Borrar un perfil |
| `sudo nmtui` | Configuración en menús de texto |
| `/etc/NetworkManager/system-connections/*.nmconnection` | Dónde viven los perfiles en RHEL 9 |
| `ssh -p PUERTO usuario@host [comando]` / `ssh -v` | Conectar, ejecutar un comando remoto, depurar |
| `ssh-keygen -t ed25519` / `ssh-copy-id [-p P] usuario@host` | Crear par de claves / instalar la pública |
| `ssh-keygen -R host` / `ssh-keygen -lf clave.pub` | Borrar huella vieja / ver huella |
| `~/.ssh/config` (`Host`, `HostName`, `Port`, `User`, `IdentityFile`) | Alias de conexión |
| `scp [-P P] origen destino` / `sftp host` | Copiar archivos por SSH / sesión interactiva |
| `rsync -avz --delete origen/ destino/` | Sincronización incremental (local o por SSH) |
| `ssh -N -L LOCAL:localhost:REMOTO host` | Túnel local a un puerto interno del servidor |
| `sudo sshd -T` / `sudo sshd -t` | Configuración efectiva de sshd / validar sintaxis |
| `journalctl -u NetworkManager` | Qué hizo NetworkManager y cuándo |
| `sudo tcpdump -i IF -n [filtro] -c N` | Ver si el tráfico llega (`icmp`, `port 22`, `udp port 53`) |
| `sudo firewall-cmd --list-all` | Qué deja pasar el firewall (Día 8) |

---

## Notas para el instructor

### Preparar antes de la clase

- Restaurar el snapshot `dia04-fin` en la VM del instructor y recorrer **todos** los labs de principio a fin, cronometrando el Bloque 3 (es el más largo y el que más se atasca).
- Crear la red Host Only en UTM y anotar el rango real que asigna macOS (`ifconfig | grep -A4 bridge1`). Tener preparada la frase "en mi Mac la red host-only es 192.168.X.0/24; en VirtualBox es 192.168.56.0/24".
- Confirmar que la VM de UTM tiene el adaptador 1 en *Emulated VLAN* con los port forwarding 2222→22 y 8080→80 (el modo *Shared Network* no permite port forwarding, y la clase asume que existe).
- Probar `romper-red.sh` en los dos modos, incluida la falla `gateway` estando conectado por SSH NAT, para saber cuánto se congela la sesión y confirmar que se recupera. Dejar el script listo para compartir (pegar en el chat o `scp`).
- Tener un `~/.ssh/config` de ejemplo listo para pegar, en versión macOS y Windows.
- Verificar que en la VM `python3 -m http.server 8000` funciona y que el túnel `ssh -N -L 8081:localhost:8000` muestra el listado en el navegador.
- Pedir a los participantes con Windows que confirmen antes de la clase `ssh -V` en PowerShell y que localicen `C:\Users\SU_USUARIO\.ssh\`.
- Precalentar la caché de `dnf` en la VM del instructor con `sudo dnf install -y bind-utils traceroute tcpdump rsync NetworkManager-tui ipcalc` para no esperar durante la demostración, y comprobar aparte `sudo dnf install -y mtr` (⚠️ Verificar en la VM: `mtr` e `ipcalc` pueden no estar en los repositorios habilitados; si faltan, se sustituye `ipcalc` por el cálculo a mano y `mtr` por `tracepath`).
- Tener a mano la salida real de `nmcli device status` y `ip -br a` de la VM UTM para mostrar la diferencia de nombres de interfaz desde el primer minuto.
- Comprobar `rpm -q NetworkManager-config-server` en la VM del curso: decide si al añadir la tarjeta aparece `Wired connection 1` (Lab 3.1 paso 3) o queda `disconnected`. Ajustar lo que se dice en clase a lo que realmente ocurre.
- En UTM (Emulated VLAN), confirmar que `ping 10.0.2.2` y `ping 8.8.8.8` responden desde la VM y qué muestra `tracepath -n 8.8.8.8`; el NAT de QEMU no siempre se comporta como el de VirtualBox con ICMP.
- Verificar tras `nmcli con down lab` qué queda en `ip -br a show enp0s8` (solo `fe80::`, nada, o `DOWN`), y el valor de `GENERAL.IP4-CONNECTIVITY` en `nmcli -f all device show enp0s8`, para que la salida esperada del Lab 5.1 coincida con la real.

### Qué estudiar si es nuevo en RHEL

1. **Modelo de NetworkManager (device vs connection, keyfiles).** Leer `man nmcli-examples` (ejemplos 1–10) y `man nm-settings-nmcli` en las secciones `ipv4` e `ipv6`. Practicar: crear un perfil, modificarlo, ver cómo cambia el `.nmconnection` con `sudo cat`, borrarlo, y leer la sección `migrate` de `man nmcli` para saber qué responder si alguien menciona `ifcfg`.
2. **Cómo NM construye `/etc/resolv.conf`.** `man NetworkManager.conf`, sección `dns=` y `rc-manager=`. Probar `ipv4.ignore-auto-dns yes`, `ipv4.dns-priority` y ver el efecto en `resolv.conf`. Saber que RHEL 9 no usa `systemd-resolved` por defecto.
3. **`sshd_config.d/` y crypto-policies.** `ls /etc/ssh/sshd_config.d/`, `cat 50-redhat.conf`, `update-crypto-policies --show`, `sudo sshd -T | grep -iE 'permitrootlogin|passwordauthentication|pubkey'`. Entender por qué una directiva en `sshd_config` puede no tener efecto si el drop-in la define antes.
4. **`ss` y `tcpdump` con filtros.** `man ss` (opciones `-t -u -l -p -n -a`, filtro `sport = :22`), `man pcap-filter` (expresiones `host`, `port`, `icmp`, `and`/`or`). Practicar capturar con `-c` para no inundar la terminal.
5. **Diagnóstico con `journalctl -u NetworkManager`.** Provocar `nmcli con down/up` y `nmcli device disconnect` y leer las líneas resultantes para reconocerlas en clase.

### Errores frecuentes de los participantes y cómo resolverlos

| Síntoma | Causa | Solución |
|---|---|---|
| `Error: Connection activation failed: No suitable device found for this connection (device enp0s3 not available because profile is not compatible with device (mismatching interface name))` | El `ifname` del perfil no coincide con la tarjeta real (UTM usa `enp0s2`, no `enp0s8`) | `nmcli device status` para ver el nombre; `sudo nmcli con mod lab connection.interface-name enp0s2; sudo nmcli con up lab` |
| Cambió el DNS/IP con `nmcli con mod` y "no pasó nada" | `mod` solo edita el perfil | `sudo nmcli con up lab` (o `nmcli device reapply`) |
| Editó `/etc/resolv.conf` con `vi` y al rato volvió a lo anterior | NM lo regenera al activar cualquier conexión | Configurar DNS con `nmcli con mod ... ipv4.dns`; si el perfil es DHCP, añadir `ipv4.ignore-auto-dns yes` |
| Perdió internet después de crear `lab` | Puso `ipv4.gateway 192.168.56.1` en el perfil host-only: dos rutas por defecto | `sudo nmcli con mod lab ipv4.gateway "" ipv4.never-default yes; sudo nmcli con up lab`; verificar `ip route` |
| `enp0s8` no aparece tras encender la VM | El adaptador 2 no quedó habilitado o sin "Cable Connected" (VirtualBox); en UTM se añadió el dispositivo pero no se guardó | Apagar, revisar *Settings > Network > Adapter 2*, encender |
| Desde el host, `ssh 192.168.56.10` → `Connection timed out` | La red host-only del host es otra (por ejemplo `192.168.57.0/24` en una instalación vieja de VirtualBox) o la VM sigue con `Wired connection 1` | En el host: `ipconfig` (Windows) / `ifconfig` (Mac) para ver la IP del adaptador host-only; en la VM `nmcli device status` |
| `Error: unknown connection 'Wired connection 1'` al borrarla | NM no llegó a crearla | Ignorar |
| `ssh-copy-id` no existe en PowerShell | Windows no lo incluye | Usar la línea con `type ... \| ssh ... "cat >> ~/.ssh/authorized_keys"` del Lab 4.1; revisar luego permisos en la VM |
| Copió la clave y sigue pidiendo contraseña | Permisos de `~/.ssh` (700) o `authorized_keys` (600) incorrectos, o pegó la clave privada en vez de la `.pub` | `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys; cat ~/.ssh/authorized_keys` (debe empezar por `ssh-ed25519`); en el servidor `sudo journalctl -u sshd -n 20` muestra `Authentication refused: bad ownership or modes` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | Restauró un snapshot con otras claves de host o reinstaló | `ssh-keygen -R "[localhost]:2222"` y/o `ssh-keygen -R 192.168.56.10`, comparar huella con `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` |
| `scp: Connection refused` o se conecta al puerto 22 del propio equipo | Usó `-p 2222` (minúscula) en `scp`/`sftp` | En `scp` y `sftp` el puerto es `-P` mayúscula; mejor usar el alias de `~/.ssh/config` |
| `~/.ssh/config` no se aplica en Windows | El archivo se guardó como `config.txt` | Crear con `New-Item` y editar; comprobar con `dir $env:USERPROFILE\.ssh` |
| `rsync: command not found` | No viene instalado en Server | `sudo dnf install -y rsync` |
| `tcpdump -i enp0s3 port 22` no para de imprimir y la sesión se vuelve inusable | Retroalimentación: la salida viaja por SSH y genera más tráfico SSH | Usar `-c N`, capturar en `enp0s8` o filtrar `not port 22` |
| `firewall-cmd --list-all` → `Authorization failed` | Se ejecutó sin `sudo` | `sudo firewall-cmd --list-all` |
| El prompt sigue diciendo `rhel01` tras `set-hostname rhel01.lab.local` | bash muestra el nombre corto (`\h`) y solo se actualiza al abrir sesión | Es correcto; `hostnamectl --static` muestra el FQDN |
| `dnf` falla con `Cannot download repodata` durante el reto | Es la falla `dns` o `gateway` del reto | Diagnosticar por capas; no es un problema de suscripción |
| `getent hosts rhel01` devuelve una `fe80::...` en vez de la IPv4 | `getent hosts` pregunta primero por IPv6 y `myhostname` responde con la link-local | Es correcto; para ver IPv4 usar `getent ahostsv4 rhel01` o `hostname -I` |

### Diferencias VirtualBox (x86_64) vs UTM (aarch64)

| Aspecto | VirtualBox 7 (participantes) | UTM (instructor) |
|---|---|---|
| Interfaces | `enp0s3` (NAT), `enp0s8` (host-only) | `enp0s1`, `enp0s2` aprox.; **verificar con `nmcli device`** |
| MAC | Prefijo `08:00:27` | Prefijo `52:54:00` u otro aleatorio |
| Red NAT | `10.0.2.0/24`, gateway `10.0.2.2`, DNS `10.0.2.3`; port forwarding en *Settings > Network > Adapter 1 > Port Forwarding* | Modo *Emulated VLAN*: mismos `10.0.2.x`; port forwarding en la configuración del dispositivo de red. El modo *Shared Network* da `192.168.64.x` sin port forwarding |
| Red host-only | `192.168.56.0/24`, host = `.1`, configurable en *File > Tools > Network Manager* | Rango asignado por macOS (vmnet); leerlo con `ifconfig` (`bridge100`/`bridge101`); el host es la `.1`. macOS también reparte DHCP en esa red |
| Añadir adaptador | VM apagada; *Adapter 2 > Host-only Adapter* | VM apagada; *New... > Network > Host Only*, tarjeta `virtio-net-pci` |
| `nmcli -f all device show` | `DRIVER: e1000`, `PRODUCT: 82540EM` | `DRIVER: virtio_net` |
| `hostnamectl` | `Virtualization: oracle`, `Architecture: x86-64` | `Virtualization: qemu` (o `apple`), `Architecture: arm64` |
| Paquetes | `.el9.x86_64` | `.el9.aarch64` |
| Comandos del día | Idénticos | Idénticos; solo cambian nombres de interfaz e IP host-only |

### Preguntas probables y respuesta corta

- **"¿Por qué todas las VM tienen 10.0.2.15?"** Cada VM en NAT tiene su propio router virtual privado dentro del hipervisor; no se ven entre sí. Por eso hace falta Host-only (o Bridged) para que dos VM o el host hablen con ella directamente.
- **"¿Puedo poner gateway en el perfil host-only?"** Se puede, pero no se debe: crearía una segunda ruta por defecto hacia una red sin salida. Si hiciera falta por alguna razón, `ipv4.never-default yes` evita que compita.
- **"¿Ya no existen los archivos ifcfg?"** En RHEL 9 se leen si están (plugin `ifcfg-rh`) pero todo lo nuevo se escribe como keyfile; `nmcli con migrate` los convierte. En RHEL 10 desaparecen.
- **"¿nmcli, nmtui o editar el archivo?"** Lo que produzca el mismo resultado con menos error. `nmcli` es copiable, auditable y lo que evalúa el examen. `nmtui` es válido cuando se está en la consola sin recordar la sintaxis.
- **"¿Por qué no editar resolv.conf si es más rápido?"** Porque NM lo reescribe y el cambio se pierde sin aviso. Si un servidor necesita que NM no lo toque: `dns=none` y `rc-manager=unmanaged` en `/etc/NetworkManager/conf.d/`, pero es la excepción.
- **"¿`ip addr add` sirve para configurar una IP?"** Funciona hasta el próximo reinicio o `nmcli con up`. Para diagnóstico rápido sí; para configurar, nunca.
- **"¿ed25519 o RSA?"** ed25519 salvo que el servidor destino sea muy antiguo (RHEL 6 o dispositivos de red viejos); entonces `rsa -b 4096`.
- **"¿Qué pasa si pierdo la clave privada?"** Se genera otra y se sustituye la pública en `authorized_keys` de cada servidor. Por eso hay que llevar inventario de dónde está cada clave y borrar la vieja.
- **"¿Cómo hago que la VM sea alcanzable desde otra PC de la oficina?"** Modo Bridged en el hipervisor: la VM toma una IP de la red física. En redes institucionales conviene coordinarlo con quien administra el DHCP.
- **"¿El túnel SSH es lo mismo que una VPN?"** Es un túnel para un puerto concreto; una VPN enruta redes completas. Para una consola web puntual el túnel basta y no requiere abrir nada en el firewall.
- **"¿Cómo veo qué configuración está usando realmente sshd?"** `sudo sshd -T`; combina `sshd_config` y todo `sshd_config.d/`.
- **"¿tcpdump se puede usar en producción?"** Sí, con filtros y `-c`; sin filtros en un servidor con tráfico alto consume CPU y llena el disco si se escribe con `-w`.

### Relación con el examen RHCSA

Objetivos del EX200 (RHEL 9) que toca este día:

- **Manage basic networking:** *Configure IPv4 and IPv6 addresses* (Lab 3.2, 3.3 con `nmcli con add/mod`, `ipv6.method manual`); *Configure hostname resolution* (`/etc/hosts`, `hostnamectl set-hostname`, DNS en el perfil); *Configure network services to start automatically at boot* (`connection.autoconnect`, y `systemctl enable` visto el Día 4); *Restrict network access using firewall-cmd/firewall* se cubre el Día 8.
- **Operate running systems:** *Access remote systems using SSH*; *Securely transfer files between systems* (`scp`, `sftp`, `rsync`).
- **Manage security:** *Configure key-based authentication for SSH* (Lab 4.1).

Cómo lo pide el examen: típicamente "configure la interfaz X con la IP estática A/24, gateway G, DNS D, que persista al reiniciar" y "el sistema debe resolver el nombre `servidorX` a la IP Y". La respuesta esperada es exactamente la secuencia `nmcli con mod ... ; nmcli con up ...` y una línea en `/etc/hosts`. Recordar a los participantes que en el examen **hay que reiniciar y comprobar**: una IP puesta con `ip addr add` vale cero puntos.

---

## Tarea y preparación para el día siguiente

1. **Reiniciar la VM y comprobar persistencia** (5 min): `sudo reboot`; al volver, `ip -br a` debe mostrar `192.168.56.10/24` en `enp0s8`, `ip route` un solo `default via 10.0.2.2`, `getent hosts servidor-nfs` la IP host-only, y `ssh rhel01 hostname` desde el equipo propio debe entrar sin contraseña. Si algo falla, es la tarea perfecta para aplicar el método por capas.
2. **Dejar el hostname corto antes del snapshot:** `sudo hostnamectl set-hostname rhel01`. El material de los días siguientes muestra `rhel01` en prompts, logs y salidas; el FQDN `rhel01.lab.local` sigue resolviendo por la línea de `/etc/hosts` (`getent hosts rhel01.lab.local`), que es lo que importa. Quien prefiera conservar el FQDN puede hacerlo: solo cambia lo que muestran `hostname` y `journalctl`.
3. **Snapshot `dia05-fin`** con la VM apagada (`sudo poweroff`) y la red funcionando. No saltarse este paso: el Día 6 modifica discos. Ojo: el snapshot guarda también la configuración de almacenamiento, así que si más adelante hay que restaurarlo, los discos que se añadan en el punto 4 desaparecen y hay que volver a añadirlos (el Día 6 lo contempla).
4. **Añadir dos discos de 5 GB** para el Día 6 (LVM y sistemas de archivos), con la VM apagada:
   - VirtualBox: *Settings > Storage > Controller: SATA > icono "Add hard disk" > Create > VDI > Dynamically allocated > 5 GB*. Repetir para el segundo. Al arrancar aparecerán como `/dev/sdb` y `/dev/sdc`.
   - UTM: *Settings > New... > Drive > Interface VirtIO, Size 5 GB*. Repetir. Aparecerán como `/dev/vdb` y `/dev/vdc`.
   - Verificar al encender: `lsblk` debe listar los dos discos nuevos de 5G sin particiones. **No formatearlos ni tocarlos**: eso es el Día 6.
5. **Practicar 20 min** (sin mirar el material): añadir una tercera IP `192.168.56.12/24` al perfil `lab` con `+ipv4.addresses`, aplicar, verificar con `ip -br a`, quitarla con `-ipv4.addresses` y volver a aplicar. Luego `dig +short` de tres dominios distintos y `dig -x` de una de las IPs obtenidas. Después, `sudo tcpdump -i enp0s8 -n icmp -c 4` mientras se hace `ping` desde el equipo propio. Y para cerrar, mirar (sin cambiar nada) la configuración efectiva del servidor SSH: `ls /etc/ssh/sshd_config.d/` y `sudo sshd -T | grep -iE 'permitrootlogin|passwordauthentication|pubkeyauthentication'`; se usará el Día 8.
6. **Lectura corta:** `man nmcli-examples` (ejemplos 1 a 5) y `man ssh_config` (buscar `Host`, `HostName`, `Port`, `IdentityFile`).
7. Anotar cualquier comando del día que no haya funcionado igual que en el material para revisarlo en el repaso del Día 6.
