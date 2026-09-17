# Comandos — SELinux: corregirlo con `semanage` (guía del instructor)

Diez minutos, después del descanso. Tres cosas que tienen que quedar: (1) `semanage port -a` para un puerto nuevo; (2) `semanage fcontext -a` + `restorecon -Rv` para una carpeta propia, **siempre los dos pasos**, y `chcon` no es solución; (3) un AVC se lee mirando `tcontext` y `tclass`, y la tabla de decisión dice qué hacer. Los cuatro labs practican exactamente eso.

**Qué es `semanage`:** "SELinux manage": la herramienta para agregar reglas locales a la política sin reescribirla. Viene en el paquete `policycoreutils-python-utils` (instalado en el Lab 1.1). Tiene sub-comandos: `port`, `fcontext`, `boolean`, `permissive`. Siempre con `sudo`.
**Qué es persistente:** que sobrevive al reinicio y a un reetiquetado del disco. Todo lo de `semanage` lo es. `chcon` y `setsebool` sin `-P`, no.
**Qué es un reetiquetado:** volver a aplicar la tabla de la política a todo el disco (o a una carpeta). Pasa al reactivar SELinux, en algunas actualizaciones, o cuando alguien corre `restorecon -R`.
**Qué es `semanage port`:** la lista de qué puerto tiene qué tipo. `-l` lista; `-a` agrega; `-m` modifica uno que ya tiene tipo; `-d` borra; `-C` ("customized") muestra solo lo tuyo. `-t` el tipo, `-p` el protocolo (tcp/udp), al final el número.
**Qué es `already defined`:** el error de `-a` cuando el puerto ya tiene un tipo (el 8080 es `http_cache_port_t`; el 8082 del reto es `us_cli_port_t`). Ahí va `-m`.
**Qué es `http_cache_port_t`:** el tipo de los puertos de proxy web (8080). La política deja que Apache los use también: por eso los tutoriales con 8080 no fallan. Usamos el 82 justamente porque falla.
**Qué es `reserved_port_t`:** el tipo de cualquier puerto menor a 1024 que no tenga dueño asignado. Solo root puede abrirlos, y además SELinux exige que el dominio tenga permiso.
**Qué es `-w` de grep:** palabra completa (Día 2). `grep -w http_port_t` no trae `pegasus_http_port_t`, que también contiene ese texto.
**Qué es `semanage fcontext`:** "file context": la tabla ruta → tipo que consulta `restorecon`. `-a -t TIPO "RUTA"` agrega una fila; `-l -C` muestra las tuyas; `-d` borra.
**Qué es `(/.*)?`:** la forma en que la tabla dice "esta carpeta y todo lo que tenga adentro". Se escribe siempre igual, pegado a la ruta y entre comillas dobles. No hay que entenderla, hay que copiarla. Sin las comillas, la shell interpreta el `?` y el `(` y el comando falla.
**Qué es `restorecon -Rv`:** aplicar la tabla. `-R` a la carpeta y todo su contenido; `-v` mostrar cada cambio (`Relabeled ... from ... to ...`). Si no imprime nada, ya estaba bien.
**Qué es `chcon`:** "change context": cambia la etiqueta de un archivo directamente, sin tocar la tabla. Funciona hasta el próximo `restorecon`, que la vuelve a poner como dice la tabla. Sirve para probar; en producción es un parche que alguien va a pisar.
**Qué es un booleano:** un interruptor sí/no de la política: partes que vienen apagadas y se prenden sin escribir reglas. `getsebool` lee; `setsebool -P` escribe persistente (la `P`); `semanage boolean -l -C` lista los que cambiaste.
**Qué es `httpd_can_network_connect`:** el booleano que deja a Apache abrir conexiones hacia otros servidores. Hace falta cuando Apache es proxy inverso.
**Qué es un proxy inverso:** un Apache que recibe el pedido del usuario y lo reenvía a otra aplicación (otro puerto u otro servidor) y devuelve la respuesta. Para eso necesita "salir": el booleano.
**Qué es `httpd_enable_homedirs`:** el booleano que deja a Apache leer la carpeta `public_html` del home de cada usuario (páginas personales). Solo se nombra.
**Qué es un AVC:** "Access Vector Cache", el nombre técnico del registro que deja SELinux por cada denegación. Decir "denegación" en clase; "AVC" es lo que hay que escribir en `ausearch -m AVC`.
**Qué es auditd:** el servicio de auditoría del kernel (Día 4: `/var/log/audit/audit.log`). SELinux le manda cada denegación. Está siempre corriendo.
**Qué es `ausearch`:** buscar en el log de auditd. `-m AVC` = solo denegaciones de SELinux; `-ts recent` = "time start: recent", los últimos 10 minutos. Con `sudo`. Cada resultado trae tres o cuatro líneas; la que importa es la que empieza con `type=AVC`, que es la última: por eso `| tail -1`.
**Qué es `{ getattr }`, `{ read }`, `{ name_bind }`, `{ name_connect }`:** la operación negada. `getattr`: leer los atributos del archivo (tamaño, fechas): lo primero que hace Apache antes de leerlo, por eso aparece más que `read`. `name_bind`: reservar un puerto para escuchar. `name_connect`: conectarse a un puerto de otro.
**Qué es `comm=`:** el nombre del programa (`httpd`).
**Qué es `scontext=` / `tcontext=`:** "source context" y "target context": la etiqueta del que intentó y la etiqueta de la cosa. En `tcontext` está la pista casi siempre: un tipo que no corresponde al lugar.
**Qué es `tclass=`:** la clase de la cosa: `file`, `dir`, `tcp_socket` (un puerto).
**Qué es `path=` / `src=`:** el archivo, o el puerto (`src=82`).
**Qué es `permissive=0` / `1`:** si el sistema estaba bloqueando (`0`) o solo registrando (`1`) cuando pasó.
**Qué es setroubleshoot:** el servicio que lee los AVC y los traduce a una frase en inglés con sugerencias. Paquete `setroubleshoot-server`, instalado en el Lab 1.1. Escribe en el journal con la etiqueta `setroubleshoot`: por eso `journalctl -t setroubleshoot` (Día 4: `-t` filtra por etiqueta).
**Qué es `sealert`:** el comando de setroubleshoot. `sealert -l CODIGO` muestra el informe de un problema.
**Qué es el CODIGO:** el identificador largo con guiones que setroubleshoot le pone a cada problema (`4c1f7a2e-...`; técnicamente un UUID). Está al final de la línea del journal. Cada VM tiene el suyo.
**Qué es la confianza (`confidence`):** el porcentaje con que cada sugerencia del informe cree ser la correcta. La de arriba, 99.5, es la buena.
**Qué es `catchall` / `audit2allow`:** la última sugerencia del informe, con 1.5 de confianza, que **siempre** aparece: fabricar una regla nueva que permita exactamente lo que falló. Casi nunca es la solución: permitiría a Apache leer archivos de `/root` para siempre. No lo usamos.
**Qué es `semanage permissive -a httpd_t`:** poner en permissive **un solo dominio**: Apache deja de ser bloqueado (y sigue registrando) mientras el resto del sistema sigue en enforcing. La alternativa profesional a `setenforce 0` cuando hay que seguir operando. Se quita con `-d`.

---

## Puertos

**Qué decir:** "cada puerto tiene un tipo. Apache solo puede escuchar en `http_port_t`. El 82 no tiene tipo: hay que dárselo. Un comando."
**Qué señalar:** en el `grep -w http_port_t`, los puertos de fábrica (80, 443, 8008…): el 82 no está. `-a` agrega; si dice `already defined`, `-m`. Lo van a ver con el 8080 en el Lab 3.1 y lo van a necesitar en el reto.

---

## Carpetas

Leer el diagrama de arriba abajo. **La frase que hay que decir:** *"Dos pasos, siempre: primero la regla en la tabla (`semanage fcontext -a`), después aplicarla (`restorecon -Rv`). Si hacés solo el segundo, no cambia nada porque la tabla no sabe de tu carpeta. Si hacés `chcon`, cambia hoy y mañana un `restorecon` lo deshace."*
**Qué señalar:** la parte `(/.*)?`: "se copia tal cual, entre comillas; significa 'la carpeta y todo lo de adentro'".

---

## Booleanos

**Qué decir:** "interruptores. Prender uno es un comando con `-P`. Apagarlos cuando no se usan: cada uno que está prendido es un permiso más."
Señalar solo `httpd_can_network_connect`: el caso real más frecuente (Apache como proxy inverso devuelve 503 hasta prenderlo).

---

## Leer una denegación

Leer el ejemplo en voz alta con la tabla de campos: "`httpd` (`comm`) intentó `getattr` sobre un `file` (`tclass`) que tiene etiqueta `admin_home_t` (`tcontext`), y el archivo es `/web/config.txt` (`path`). Un archivo de /root dentro de la web."
**Qué señalar:** las tres herramientas van de crudo a traducido: `ausearch` siempre está; `journalctl -t setroubleshoot` da la frase y el código; `sealert -l` el informe.

---

## Tabla de decisión

**Qué decir:** "esta tabla es el día de hoy en cinco filas. Con `tcontext` y `tclass` se cae en una fila, y la fila dice qué comando. Imprímanla."
**Qué señalar:** lo que **no** está: `setenforce 0` como solución, `disabled`, `chcon`. "Si la respuesta que se te ocurre es una de esas tres, está mal."

---

## Permissive para un solo servicio

Una frase: "cuando una aplicación nueva da problemas y no podés parar, esto en vez de `setenforce 0`: solo ese servicio queda sin bloqueo, todo lo demás sigue protegido, y se quita cuando se resolvió." Lo hacen en el Lab 3.4, Parte 5.
