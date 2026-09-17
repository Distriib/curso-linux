# 1 — Conceptos de red: los cinco números (guía del instructor)

Diez minutos, tipeando mientras explicás. Tiene que quedar: (1) una IP sola no dice nada, hace falta la máscara; (2) el gateway es la salida y está en `ip route` como `default via`; (3) los nombres se buscan primero en `/etc/hosts` y después en el DNS de `/etc/resolv.conf`, que lo escribe NetworkManager; (4) el puerto es el servicio dentro de la máquina. Lo demás se ve con las manos en el Bloque 2. No leer las tablas enteras.

**Qué es una dirección IP (IPv4):** cuatro números de 0 a 255 separados por puntos. Identifica una máquina dentro de una red.
**Qué es la máscara:** dice qué parte de la IP es "la red" (el barrio) y qué parte es "la máquina" (la casa). `255.255.255.0` = los tres primeros números son la red, el cuarto es la máquina.
**Qué es CIDR:** la forma corta de escribir la máscara: `/24` = 24 bits de red = `255.255.255.0`. Cada número de la IP son 8 bits; `/24` son tres números completos.
**Qué es un bit:** un 0 o un 1. Cada número de la IP (0 a 255) se guarda en 8 bits. No hace falta convertir nada hoy: `ipcalc` lo hace.
**Qué es la red y el broadcast:** la primera dirección del rango (`.0`) es el nombre de la red; la última (`.255`) es el broadcast, "para todos". Ninguna se asigna a una máquina.
**Qué es un host:** cualquier máquina con IP en la red.
**Qué es el gateway:** el router de la red local, la puerta de salida. Cuando el destino no está en mi red, le entrego el paquete al gateway.
**Qué es la ruta por defecto:** la línea `default via ...` de `ip route`: "todo lo que no sé a dónde va, al gateway".
**Qué es `ip route`:** el comando que muestra la tabla de rutas: por dónde sale cada destino.
**Qué es DNS:** el servicio que traduce nombres (`redhat.com`) a IPs. "Resolver" un nombre es hacer esa traducción.
**Qué es `/etc/hosts`:** archivo local con líneas `IP nombre`. Se consulta antes que el DNS.
**Qué es `/etc/resolv.conf`:** archivo con la línea `nameserver IP`: a qué servidor DNS preguntar. En RHEL 9 lo genera NetworkManager.
**Qué es `/etc/nsswitch.conf`:** archivo que dice en qué orden buscar. La línea `hosts: files dns myhostname` = primero `/etc/hosts`, después DNS, después el propio nombre del servidor.
**Qué es NetworkManager:** el servicio de RHEL 9 que administra la red: tarjetas, IPs, DNS. Se maneja con `nmcli`. Es el Bloque 3.
**Qué es un puerto:** un número de 0 a 65535 que identifica un servicio dentro de una máquina. El 22 es SSH, el 80 la web.
**Qué es un socket:** la pareja `IP:puerto` donde un programa escucha o por la que conversa.
**Qué es TCP y UDP:** las dos formas de mandar datos. TCP arma una conexión y garantiza que llegue todo y en orden (SSH, web). UDP manda y no espera confirmación: más rápido, sin garantía (DNS, hora).
**Qué es `ss`:** el comando que lista sockets: quién escucha en qué puerto. Reemplaza al viejo `netstat`, que en RHEL 9 no viene.
**Qué es IPv6:** la versión nueva de las direcciones, más largas, escritas en hexadecimal (números y letras a–f) separadas por `:`. Convive con IPv4.
**Qué es link-local (`fe80::`):** una dirección IPv6 que toda tarjeta se inventa sola y que solo sirve en el mismo cable. No sale a ningún lado. Siempre está; no hay que configurarla.
**Qué es un hipervisor:** el programa que corre la VM: VirtualBox en las máquinas de ellos, UTM en tu Mac.
**Qué es NAT:** el hipervisor pone a la VM detrás de un router privado propio. La VM sale a internet, pero nadie llega a ella salvo por reenvío de puertos. Toda VM en NAT tiene `10.0.2.15`.
**Qué es el reenvío de puertos (port forwarding):** la regla del hipervisor "lo que llegue al puerto 2222 de mi PC, mandalo al 22 de la VM". Por eso hasta hoy se conectan con `-p 2222`.
**Qué es Host-only:** una red privada solo entre la PC y la VM. Sin internet, pero con acceso directo por IP. En VirtualBox es `192.168.56.0/24` y la PC es la `.1`.
**Qué es Bridged:** la VM se conecta a la red física real como si fuera otra PC más. Lo más parecido a producción; en redes de oficina suele estar bloqueado.
**Qué es `ipcalc`:** una calculadora de redes: le das IP/máscara y te dice red, broadcast y máscara larga.

---

## IP y máscara

**Qué decir:** "una IP sin máscara es como una dirección sin ciudad. La máscara dice cuánto de la dirección es el barrio."
**Qué señalar:** en la tabla, la fila `/24` es la que importa hoy: red `.0`, broadcast `.255`, 254 máquinas. La fila `/26` la ven con las manos en el lab.
**Frase exacta:** *"Dos equipos se hablan directo solo si están en la misma red. Si no, necesitan un intermediario: el gateway."*

---

## Gateway y ruta por defecto

`ip route`.
**Qué señalar:** `default via 10.0.2.2`: ese es el gateway de la red NAT. La segunda línea la crea el sistema solo al tener una IP: "a `10.0.2.0/24` llego directo, sin gateway".
**Qué decir:** "sin la línea `default`, no hay internet. Con una `default` equivocada, tampoco. Es lo primero que se mira en un ticket de 'no tengo internet'."

---

## DNS: de nombre a IP

`cat /etc/hosts`, `cat /etc/resolv.conf`, `grep hosts: /etc/nsswitch.conf`.
**Qué señalar:** la primera línea de `resolv.conf`: `# Generated by NetworkManager`. En `nsswitch.conf`, `files` va antes que `dns`: `/etc/hosts` le gana al DNS (lo comprueban en el Lab 2.2).
**Frase exacta:** *"Este archivo no se edita. Si lo editás a mano, NetworkManager lo vuelve a escribir y perdés el cambio sin aviso. El DNS se cambia con `nmcli`, en el Bloque 3."*

---

## Puertos

**Qué decir:** "la IP es el edificio, el puerto es la oficina." Señalar solo 22 y 80: son los dos que van a ver escuchando en el lab (`sshd` y el `httpd` que instalaron ayer). No leer toda la tabla.

---

## IPv6 en dos líneas

Una frase: "van a ver una dirección larga que empieza con `fe80` al lado de cada IP. Es normal, la tiene toda tarjeta, no hay que hacer nada con ella."

---

## Las tres redes del hipervisor

**Qué decir:** "por eso todos tienen `10.0.2.15` y por eso se conectan con `-p 2222`. Hoy agregamos una segunda tarjeta Host-only para tener una IP fija a la que se llega directo."
**Qué señalar:** la fila Host-only dice "no sale a internet": por eso en el Bloque 3 **no** se le pone gateway al perfil nuevo. Tu Mac con UTM usa otro rango para Host-only (lo decide macOS, por ejemplo `192.168.64.x`): decilo una vez acá, y que donde el material dice `192.168.56.10` vos vas a mostrar tu IP y ellos usan la del material.

---

## Calcular una red

Solo mostrar el comando y las tres letras. Se practica en el lab.
