# Comandos — firewalld (guía del instructor)

Quince minutos, tipeando mientras explicás. Antes de arrancar: pediles que lancen ya el `dnf install` de la Parte 1 del Lab 1.1 (tarda dos o tres minutos; descarga mientras hablás). Tres cosas que tienen que quedar: (1) vos hablás con `firewall-cmd`, `firewall-cmd` habla con firewalld, y firewalld escribe reglas en el kernel; (2) el tráfico cae en una zona según de dónde viene, y **el origen manda sobre la interfaz**; (3) sin `--permanent` se pierde, con `--permanent` no se aplica hasta `--reload`. Lo demás se practica en los dos labs.

**Qué es un firewall:** un filtro que decide qué conexiones de red entran al servidor (y cuáles no), según reglas. Hoy hablamos del firewall del propio servidor, no del de la institución.
**Qué es un puerto abierto:** un puerto (Día 5: el número que identifica a un servicio dentro de la máquina; 22 SSH, 80 web) al que el firewall deja llegar conexiones. "Abrir el 80" = dejar pasar hacia el puerto 80.
**Qué es nftables:** el mecanismo del kernel que filtra paquetes en RHEL 9. Reemplazó a iptables. Nadie lo escribe a mano cuando hay firewalld: el próximo `--reload` lo pisa.
**Qué es firewalld:** el servicio (daemon, Día 4) que traduce reglas humanas (zonas, servicios, puertos) a reglas de nftables y las mantiene. Es una unidad de systemd como cualquier otra: `firewalld.service`.
**Qué es `firewall-cmd`:** el comando para hablarle a firewalld. Siempre con `sudo`.
**Qué es una zona:** un nivel de confianza con su lista de servicios y puertos permitidos. Cada paquete que llega se clasifica en una zona y se evalúa con las reglas de esa zona.
**Qué es la interfaz:** la tarjeta de red (Día 5): `enp0s3` es la NAT y `enp0s8` la host-only en VirtualBox. Si tu VM corre en UTM se llaman distinto (`enp0s1`, `enp0s2`): en tu pantalla salen esos nombres y en la de ellos los otros. Decilo antes de que pregunten.
**Qué es el origen (`source`):** la red IP de la que viene el paquete, por ejemplo `192.168.56.0/24`. Si esa red está asignada a una zona, el paquete va a esa zona **aunque** la interfaz por la que entró esté en otra. Eso es "el origen manda".
**Qué es `/24`:** la máscara (Día 5): los primeros tres números son la red. `192.168.56.0/24` = todas las direcciones `192.168.56.x`.
**Qué es la zona por defecto:** la que recibe todo lo que no coincide con ningún origen ni interfaz asignada; y la que usan los comandos cuando no ponés `--zone`. En RHEL es `public`.
**Qué es `target`:** qué hace la zona con lo que **no** está en su lista. `default` = rechaza avisando; `DROP` = tira sin responder; `ACCEPT` = deja pasar todo.
**Qué es un servicio de firewalld:** un archivo de texto que agrupa puertos bajo un nombre: `http` es `80/tcp`. No es lo mismo que un servicio de systemd (`httpd.service` es el programa; `http` es la puerta), aunque los nombres se parezcan. Van a confundirlos: decir "servicio del firewall" y "servicio de systemd".
**Qué es XML:** un formato de texto con etiquetas entre `<` y `>`. Solo hay que saber leerlo: `<port protocol="tcp" port="80"/>` = puerto 80 TCP.
**Qué es `cockpit`:** la consola web de administración de RHEL (puerto 9090). Viene abierta en el firewall aunque el paquete no esté instalado. Algunos la activaron en la tarea del Día 7.
**Qué es `dhcpv6-client`:** las respuestas del servidor DHCP de IPv6. Viene abierto de fábrica; inofensivo, no se toca.
**Qué son `mdns` y `samba-client`:** descubrimiento de equipos en la red local y cliente de archivos compartidos de Windows. Vienen abiertos en `internal` de fábrica. No importan hoy: no explicarlos más que con esa frase.
**Qué es runtime:** la configuración que está corriendo ahora. Un cambio sin `--permanent` se aplica al instante y **se pierde** con `--reload` o al reiniciar.
**Qué es permanent:** lo que se escribe en `/etc/firewalld/`. Sobrevive, pero **no se aplica** hasta `--reload`.
**Qué es `--reload`:** recargar: leer lo permanente y aplicarlo. Lo que era solo runtime desaparece. No corta las conexiones ya establecidas (tu SSH sigue).
**Qué es `--runtime-to-permanent`:** copiar a disco lo que está corriendo. Es la forma "probar primero, guardar después".
**Qué es `--query-service`:** una pregunta sí/no: responde `yes` o `no`. Sirve en scripts (Día 7).
**Qué es `wc -w`:** contar palabras (Día 2 vieron `wc -l`, líneas). `--get-services` imprime todos los nombres en una sola línea separados por espacios: `wc -w` los cuenta.
**Qué es `nft`:** el comando de nftables. `nft list ruleset` muestra las reglas reales que firewalld escribió. Solo se mira.
**Qué es NAT / port forwarding:** la ruta por la que entran con `ssh -p 2222 localhost` (Días 1 y 5): el hipervisor recibe en un puerto de tu computadora y lo reenvía a la VM. La VM ve que el paquete viene de `10.0.2.2`, que no está asignado a ninguna zona: cae en `public`.
**Qué es host-only:** la segunda red (Día 5), `192.168.56.0/24`, directa entre la computadora del participante (`.1`) y la VM (`.10`). Hoy la declaramos "red de administración". Si tu VM es UTM, tu red host-only es otra: usá la tuya y avisá que ellos usan la del archivo.

---

## Por qué empezamos por el firewall

**Qué decir:** "Cuando alguien escanea un servidor, lo primero que obtiene es la lista de puertos abiertos. Cada uno es una puerta abierta las 24 horas. Si el servicio de atrás tiene una vulnerabilidad, esa es la entrada. Hoy decidimos nosotros qué puertas quedan."
**Qué señalar:** los tres que vienen abiertos de fábrica: `ssh` (lo necesitamos), `cockpit` (nadie lo pidió: lo cerramos en el Lab 4.3), `dhcpv6-client` (inofensivo).

---

## Quién hace qué

Los tres comandos de consulta: `--state`, `--get-default-zone`, `--list-all`.
**Qué decir:** "nftables es la cerradura; firewalld es el portero que tiene la lista; `firewall-cmd` es cómo le hablás al portero."
**Qué señalar:** en `--list-all`, la línea `services: cockpit dhcpv6-client ssh` y la línea `ports:` vacía. `http` no está: por eso en el Lab 1.1 la web no va a cargar desde afuera. Las dos carpetas: `/etc/firewalld/` es tuya; `/usr/lib/firewalld/` es de fábrica y no se edita.

---

## Zonas

**La frase que hay que decir despacio:** *"Un paquete que llega se clasifica primero por de dónde viene (la red de origen), y solo si eso no decide, por la interfaz por la que entró. El origen manda sobre la interfaz."* Es la fuente número uno de confusión y lo van a ver con las manos en el Lab 1.2, Parte 3: la web va a dejar de cargar por una ruta y no por la otra.
**Qué señalar en la tabla:** solo tres filas: `public` (donde estamos), `internal` (donde va a ir la red host-only), `trusted` (todo abierto: **nunca**). `drop` y `block` se nombran y nada más.

---

## Servicios predefinidos

`--get-services | wc -w`, `--info-service=http`, `cat` del XML.
**Qué decir:** "un servicio del firewall es solo un nombre para uno o más puertos. `http` es 80. Nada más que eso."
**Qué señalar:** en el XML, la línea `<port protocol="tcp" port="80"/>`. Si no hay servicio para lo que querés (el 82 de hoy), se abre el puerto directo con `--add-port=82/tcp`.

---

## Runtime vs permanent

**Esto es lo que más se rompe. Decirlo dos veces:** *"Sin `--permanent`, el cambio se aplica ya y se pierde al recargar o reiniciar. Con `--permanent`, se guarda en disco y no se aplica hasta que hacés `--reload`."*
**Qué señalar:** las dos formas correctas de la lista. La tercera (hacer el comando dos veces) funciona pero se olvida; no la enseñamos. El Lab 1.1 hace las cosas mal a propósito (Parte 5) y después bien (Parte 6).

---

## Anatomía

Leer el diagrama de derecha a izquierda: qué abrís, en qué zona, si se guarda. **Qué decir:** "sin `--zone`, el comando trabaja sobre `public`. Cuando la red host-only esté en `internal`, cada cosa que publiquemos se abre dos veces: una en cada zona."

---

## Consultar / Cambiar

No leer las tablas. Señalar cuatro: `--get-active-zones` (qué zonas están en uso y por qué), `--list-all` (todo de una zona), `--add-service` con `--permanent`, y `--reload`. El resto lo van a tipear en el lab.
