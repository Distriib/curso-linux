# 2 — Logs: quién guarda qué (guía del instructor)

Ocho minutos. Tienen que quedar dos cosas: (1) hay **cuatro** programas de log y cada uno hace un trabajo distinto (la tabla de arriba, con la analogía del edificio); (2) `journalctl` filtra por **campos**, no solo por texto. Lo demás (reglas de rsyslog, colector, logrotate, auditd) se lee por encima: cada uno tiene su lab. Tipeás los cinco comandos del primer bloque bash mientras explicás la tabla; el resto no se demuestra acá.

**Qué es journald:** el servicio `systemd-journald`. Recibe lo que escriben todos los servicios, el kernel y los programas que usan syslog, y lo guarda en un formato binario con índices. Lo consultaron todo el curso con `journalctl` (Día 4).
**Qué es persistente (en el journal):** que sobrevive al reinicio. Si existe la carpeta `/var/log/journal/`, journald guarda ahí (disco); si no, guarda en `/run/log/journal` (memoria) y se pierde al apagar. En RHEL 9 la carpeta existe: el journal es persistente de fábrica.
**Qué es rsyslog:** el servicio que escribe los archivos de texto de `/var/log/` (`messages`, `secure`, `cron`, `maillog`) leyendo del journal. Lo vieron el Día 4 con la regla `local0`. También manda logs a otro servidor y recibe de otros.
**Qué es syslog:** el protocolo clásico de logs: cada mensaje lleva una **facilidad** (quién habla) y una **prioridad** (qué tan grave). rsyslog y `logger` hablan syslog.
**Qué es auditd:** el servicio de auditoría del kernel. No registra "lo que los programas cuentan" sino "lo que el kernel vio hacer": quién abrió tal archivo, quién ejecutó tal comando. Ahí caen también las denegaciones de SELinux (AVC, Día 8). Guarda en `/var/log/audit/audit.log`, con fechas en segundos desde 1970: por eso no se lee con `grep`, se consulta con `ausearch`.
**Qué es logrotate:** el programa que cada noche renombra el log activo (`.1`), comprime los viejos y borra los que pasan de la cantidad indicada. Sin él, `/var/log` llena el disco.
**Qué es un campo del journal:** cada mensaje se guarda con metadatos: `_PID` (proceso), `_UID` (usuario), `_COMM` (nombre del programa), `_SYSTEMD_UNIT` (unidad), `PRIORITY`, `SYSLOG_IDENTIFIER` (la etiqueta). Los que empiezan con `_` los pone journald y no se pueden falsificar; los otros los pone el programa.
**Qué es `-u` vs `_SYSTEMD_UNIT=`:** `-u httpd` es un atajo que además incluye lo que systemd escribe **sobre** la unidad (`Started`, `Stopped`); el campo directo trae solo lo que la unidad escribió. Por eso `-u` muestra más.
**Qué es `-k`:** solo mensajes del kernel. Es `dmesg`, pero con fecha y con acceso a arranques anteriores (`-k -b -1`), cosa que `dmesg` no puede.
**Qué es una etiqueta (`-t`):** el nombre que un programa pone delante de sus mensajes (`sudo[1234]:`, `backup[...]:`). `logger -t backup "texto"` manda un mensaje con esa etiqueta; `journalctl -t backup` los busca. `logger` lo usaron el Día 4 y el Día 7.
**Qué es `-o verbose` / `-o json-pretty` / `-o short-iso`:** formatos de salida: todos los campos uno por línea / en JSON (para un script o un SIEM) / con la fecha en formato ISO (`2026-09-16T08:01:20-0500`).
**Qué es un SIEM:** el sistema central que junta los logs de todos los servidores de una institución y busca patrones. Recibe por syslog o JSON.
**Qué es `--disk-usage` / `--vacuum-time=2weeks`:** cuánto ocupa el journal / borrar los archivos **archivados** (ya cerrados) de más de dos semanas. Nunca toca el archivo activo.
**Qué es un drop-in (`journald.conf.d/`):** un archivo propio en una carpeta `.conf.d/` que se suma a la configuración del sistema sin editar el archivo original. Es la costumbre en RHEL 9: sobrevive a las actualizaciones. Lo vieron con `sshd_config.d/` el Día 8.
**Qué es `Storage=persistent` / `SystemMaxUse=500M` / `MaxRetentionSec=1month`:** guardar en disco siempre (aunque la carpeta no exista, la crea) / tope de tamaño / borrar lo de más de un mes.
**Qué es una facilidad:** de dónde viene el mensaje: `auth` (autenticación), `cron`, `kern` (kernel), `mail`, `daemon` (servicios), `user`, y `local0` a `local7`, que son libres para las aplicaciones propias. El Día 4 usaron `local0`; hoy `local5` para no pisarla.
**Qué es una prioridad:** la gravedad, de `debug` (menor) a `emerg` (mayor). En una regla, `local5.err` significa "err **y las más graves**" (crit, alert, emerg).
**Qué es `stop` (en rsyslog):** "este mensaje ya está atendido, no sigas evaluando reglas". Sin él, un mensaje `local5.info` también cumple la regla `*.info` de `/etc/rsyslog.conf` y aparece duplicado en `messages`. El Día 4 lo escribieron como `& stop` (`&` = "la misma condición de la línea anterior"); `local5.* stop` es lo mismo, explícito.
**Qué es `/etc/rsyslog.d/`:** la carpeta de las reglas propias, un archivo por tema. La línea `include(...)` de `/etc/rsyslog.conf` las carga **antes** de sus propias reglas.
**Qué es `rsyslogd -N1`:** validar la configuración sin arrancar (`-N1` = nivel de verificación 1). Es el `visudo -c` de rsyslog. Si hay un error de sintaxis, dice archivo y línea.
**Qué es un módulo (`module(load="imtcp")`):** rsyslog carga funciones por partes. `im` = *input module* (entrada): `imjournal` lee el journal, `imuxsock` el socket local, `imtcp` la red por TCP, `imudp` por UDP. `om` = *output module* (salida): `omfile` escribe archivos, `omfwd` reenvía.
**Qué es un `ruleset`:** un grupo de reglas con nombre. Al atar una entrada (`input`) a un ruleset, lo que llega por esa entrada se evalúa **solo** con esas reglas, no con las normales. Es lo que evita el bucle: si el servidor también es cliente (`*.* @@IP:514`), sin ruleset lo recibido volvería a salir hacia sí mismo, para siempre.
**Qué es un colector:** el servidor central que recibe los logs de los demás. En clase, la propia VM.
**Qué es 514:** el puerto clásico de syslog. Por UDP (`@IP:514`) es el histórico, sin garantía de entrega; por TCP (`@@IP:514`) no se pierden mensajes. Una `@` = UDP, dos `@@` = TCP.
**Qué es `rsh_port_t` / `syslogd_port_t`:** las etiquetas SELinux de puertos (Día 8). 514/tcp viene etiquetado para `rsh` (un programa antiguo), no para syslog; por eso rsyslog no puede escuchar ahí hasta que se cambia con `semanage port -m` (`-m` = modificar una etiqueta existente; `-a` fallaría con `already defined`).
**Qué es `logger -n IP -P 514 -T`:** mandar el mensaje directo a un servidor (`-n`), a ese puerto (`-P`), por TCP (`-T`). Sirve para probar el colector sin configurar un cliente.
**Qué es `daily` / `rotate 7` / `compress` / `delaycompress`:** rotar cada día / conservar 7 copias / comprimirlas / dejar la última rotada sin comprimir (por si el programa aún la está escribiendo).
**Qué es `missingok` / `notifempty` / `create 0640 root root`:** no dar error si el log no existe / no rotar un log vacío / crear el nuevo con esos permisos, dueño y grupo.
**Qué es `postrotate` … `endscript`:** comandos que corren después de rotar. Ahí va `systemctl reload httpd` o `logger`, para avisar al programa que abra el archivo nuevo.
**Qué es `logrotate -d` / `-f`:** `-d` (debug) simula y explica sin tocar nada; `-f` fuerza la rotación aunque no toque.
**Qué es `/var/lib/logrotate/logrotate.status`:** el archivo donde logrotate anota cuándo rotó cada log por última vez. Con eso decide si "ya toca".
**Qué es `logrotate.timer`:** el temporizador de systemd (Día 7) que ejecuta logrotate a medianoche.
**Qué es `-w RUTA -p wa -k LLAVE`:** una regla de auditd: vigilar (`-w`, *watch*) esa ruta, para escritura (`w`) y cambio de atributos (`a`: permisos, dueño), y etiquetar los eventos con una llave (`-k`) para buscarlos después.
**Qué es `/etc/audit/rules.d/` / `augenrules --load` / `auditctl -l`:** la carpeta de reglas persistentes (archivos `.rules`) / el comando que las junta y las carga / la lista de reglas cargadas ahora.
**Qué es `ausearch -k LLAVE -ts recent -i`:** buscar eventos con esa llave, desde hace 10 minutos (`-ts recent`; también `today`, `08:00`), interpretando (`-i`) números como nombres (UID → usuario, número de syscall → nombre).
**Qué es `auid`:** *audit UID*: el usuario que **inició la sesión**. No cambia con `sudo` ni con `su`: por eso la auditoría sabe que fue `student` aunque el comando corriera como root.
**Qué es `-m USER_CMD`:** el tipo de evento que auditd genera por cada comando ejecutado con `sudo`, con el comando completo.
**Qué es `aureport --summary`:** un resumen con conteos: logins, fallos, cambios de cuentas, comandos. Es el informe para un incidente.

---

## Cuatro programas, cuatro trabajos

**Qué decir:** "journald graba todo; rsyslog copia lo importante a archivos de texto y a la central; auditd graba solo las puertas que le indicás; logrotate vacía el archivero cada noche."
**Qué señalar:** en la salida de los cinco comandos: `--list-boots` con varias líneas (persistente), la carpeta con el ID de la máquina dentro de `/var/log/journal/`, los archivos clásicos en `/var/log/`, `active` dos veces, y el timer con su `NEXT` a medianoche.

---

## `journalctl` — filtrar por campo

**Qué decir:** "hasta hoy filtraron por unidad y por fecha. El journal tiene índices por proceso, por usuario, por programa. Eso es lo que lo hace rápido, y lo van a ver con `-o verbose` en el lab."
**Qué señalar:** la fila `-u` vs `_SYSTEMD_UNIT=`; y `-k`: "`dmesg` del arranque anterior, algo que `dmesg` solo no puede."

---

## rsyslog — anatomía de una regla

Leer el diagrama de izquierda a derecha: facilidad, prioridad, destino. **La frase:** "`local5.err` no es 'solo err': es 'err y peor'. Y `stop` es lo que evita que el mismo mensaje aparezca dos veces." Señalar que `@@` (TCP) es lo que se usa para un colector real.

---

## Recibir logs de otros servidores

No leer la configuración línea por línea. Decir: "la entrada TCP va atada a un `ruleset`: lo que entra por red va a su archivo y no vuelve a pasar por las reglas normales. Sin eso, un servidor que también envía se manda sus mensajes a sí mismo en bucle, y el disco se llena en minutos." Y la fila de SELinux: "514/tcp no es de syslog para SELinux; hay que decírselo."

---

## logrotate

**Qué señalar:** `delaycompress`, porque es lo que van a ver en el lab (la primera rotación no comprime). Y `-d`: "simular antes de forzar."

---

## auditd

**Qué decir:** "la diferencia con los logs normales: esto lo anota el kernel, no el programa. El programa puede olvidarse de escribir un log; el kernel no." Señalar `auid` en la tabla de comandos: "aunque hagan `sudo -i`, la auditoría sabe quién entró."
