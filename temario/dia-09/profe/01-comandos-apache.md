# Comandos — Apache: varios sitios y HTTPS (guía del instructor)

Ocho minutos, tipeando los tres comandos de "Cómo está armado" y leyendo el resto. Tres cosas que tienen que quedar: (1) un servidor sirve varios sitios y elige por el **nombre** que pidió el navegador; (2) **siempre** `configtest` antes de reiniciar; (3) HTTPS es `mod_ssl` más un certificado; el autofirmado cifra igual, pero el navegador avisa. Lo demás se practica en los dos labs.

**Qué es Apache / `httpd`:** el servidor web. El paquete y el servicio se llaman `httpd`; el programa que se configura es "Apache". Ya lo usaron los Días 6 y 8.
**Qué es una directiva:** una línea de configuración de Apache: `Listen 80`, `DocumentRoot "/var/www/html"`. Palabra clave y valor.
**Qué es `Listen`:** en qué puerto escucha. El Día 8 lo movieron al 82 y al 8082; la tarea lo devolvió al 80.
**Qué es `DocumentRoot`:** la carpeta cuyo contenido se sirve. `/var/www/html/index.html` es lo que responde `http://localhost/`.
**Qué es `conf.d/`:** la carpeta `/etc/httpd/conf.d/`. `httpd.conf` termina con `IncludeOptional conf.d/*.conf`: "cargá todo lo que haya ahí". Se cargan **en orden alfabético**: por eso los nombres empiezan con números cuando el orden importa.
**Qué es `access_log` / `error_log`:** en `/var/log/httpd/`. Cada pedido atendido va al primero; cada problema, al segundo. `/etc/httpd/logs` es un enlace a esa carpeta.
**Qué es `apachectl configtest`:** revisa la sintaxis de toda la configuración sin tocar el servicio. `Syntax OK` o el archivo y la línea del error. Es el `visudo -c` de Apache.
**Qué es un virtual host:** un sitio dentro de Apache. Un bloque `<VirtualHost>` con su nombre y su carpeta. Varios bloques = varios sitios en la misma IP y el mismo puerto.
**Qué es la cabecera `Host:`:** cuando el navegador pide `http://intranet.lab.local/`, además de conectarse a la IP manda una línea `Host: intranet.lab.local`. Apache la lee para elegir el sitio. Sin esa cabecera todos los sitios serían iguales para él.
**Qué es `ServerName` / `ServerAlias`:** el nombre principal del sitio y sus otros nombres. Se comparan con la cabecera `Host:`.
**Qué es `ErrorLog` / `CustomLog`:** un log propio para este sitio. `logs/intranet-error_log` es relativo a `/etc/httpd`, o sea `/var/log/httpd/intranet-error_log`. `combined` es el formato estándar de las líneas.
**Qué es `<Directory>`:** un bloque que dice qué se permite hacer dentro de una carpeta. `Require all granted` = cualquiera puede pedir archivos de ahí. `Options Indexes` = si no hay `index.html`, Apache arma solo una lista de los archivos. `FollowSymLinks` = seguir enlaces simbólicos. `AllowOverride None` = no leer archivos `.htaccess`.
**Qué es `default server`:** el sitio que Apache usa cuando ninguna cabecera `Host:` coincide: **el primer virtual host cargado**. No es el `DocumentRoot` global. Por eso el sitio por defecto se escribe en `00-default.conf`: el `00` lo pone primero.
**Qué es `curl -H "Host: x"`:** `-H` agrega una cabecera al pedido. Sirve para probar un sitio por nombre sin tener DNS ni tocar `/etc/hosts`.
**Qué es `apachectl -S`:** lista los virtual hosts cargados, de qué archivo salieron y cuál es el `default server`. También muestra `Main DocumentRoot`.
**Qué es `systemctl reload httpd`:** relee la configuración sin cortar conexiones. `restart` corta y arranca de nuevo. Para agregar un sitio alcanza `reload`.
**Qué es `httpd_sys_content_t`:** la etiqueta SELinux (Día 8) de lo que Apache puede leer. Todo lo que se crea dentro de `/var/www` la hereda. Fuera de `/var/www` hay que declararla con `semanage fcontext` y `restorecon`, como en el Lab 3.2 del Día 8.
**Qué es HTTPS:** HTTP cifrado. Puerto 443. El servidor presenta un **certificado** y con él se arma la conexión cifrada.
**Qué es un certificado:** un archivo que dice "esta clave pública pertenece a `rhel01`", firmado por alguien. Tiene un **sujeto** (a nombre de quién), un **emisor** (quién lo firmó) y fechas de validez. Va con una **clave privada**, que es el secreto y nunca sale del servidor.
**Qué es una autoridad certificadora (CA):** quien firma certificados. Los navegadores traen una lista de CAs en las que confían. Si el emisor no está en esa lista, el navegador avisa.
**Qué es autofirmado:** el certificado lo firmó el mismo servidor: sujeto y emisor son iguales. Cifra perfectamente; lo que falta es que alguien externo lo respalde. En una intranet se usa la CA de la institución; en Internet, Let's Encrypt, que necesita un dominio público.
**Qué es `mod_ssl`:** el módulo de Apache que habla HTTPS. Al instalarlo aparece `conf.d/ssl.conf` con `Listen 443` y las rutas del certificado y la clave.
**Qué es `/etc/pki/tls/`:** donde RHEL guarda certificados (`certs/`) y claves privadas (`private/`). PKI = infraestructura de clave pública.
**Qué es `httpd-init`:** un servicio auxiliar que corre una sola vez antes de `httpd`: si no existe `/etc/pki/tls/certs/localhost.crt`, genera un certificado autofirmado y su clave. Por eso queda `inactive (dead)`: ya hizo su trabajo.
**Qué es `openssl x509`:** lee un certificado. `-noout` = no volcar el archivo, `-subject -issuer -dates` = mostrar solo eso.
**Qué es `curl -k`:** "no validés el certificado". Solo para pruebas: es exactamente el "Continuar de todos modos" del navegador.
**Qué es `curl -I`:** pide solo las cabeceras de la respuesta (el código `200`, `403`, el `Server:`), no el contenido. Sale en el Bloque 4.
**Qué son las zonas `public` e `internal`:** las dos zonas activas del firewall (Día 8). El tráfico que entra por el port forwarding de VirtualBox cae en `public`; el que viene de la red host-only `192.168.56.0/24` cae en `internal`, porque el Día 8 esa red se puso como origen de `internal`. Un servicio abierto solo en `public` funciona desde `localhost:8080` y falla desde `192.168.56.10`. Por eso hoy todo se abre dos veces.
**Qué es `--permanent` / `--reload`:** `--permanent` guarda la regla en disco pero no la aplica; `--reload` aplica lo guardado. Sin `--permanent` se aplica ya pero se pierde al recargar (Día 8).

---

## Cómo está armado

`ls /etc/httpd/`, `ls /etc/httpd/conf.d/`, `grep -n IncludeOptional`.
**Qué decir:** "`httpd.conf` es la configuración general; `conf.d/` es un archivo por sitio; y la última línea de `httpd.conf` es la que junta las dos cosas".
**Qué señalar:** en `conf.d/` hoy hay `autoindex.conf`, `README`, `userdir.conf`, `welcome.conf`: nada nuestro todavía. La línea `IncludeOptional conf.d/*.conf`. Y la frase: *"antes de reiniciar, siempre `configtest`. Un error de sintaxis deja el servicio caído y la página fuera de línea."*

---

## Un servidor, varios sitios

No tipear nada: leer el diagrama de arriba a abajo, señalando `ServerName`, `DocumentRoot` y `Options Indexes`.
**Qué decir:** *"una recepcionista en un edificio con una sola puerta: pregunta '¿a quién busca?' y lo manda. La pregunta es la cabecera `Host:`; la respuesta la da el `ServerName`."*
**Qué señalar:** la frase en negrita de la trampa: si ningún nombre coincide, sirve **el primer virtual host cargado**. En el lab lo van a ver con `apachectl -S` (`default server rhel01`). Y la tabla: `curl` sin `-H` y con `-H` es toda la prueba del lab.

---

## HTTPS

Leer la tabla. Dos filas para detenerse: `localhost.key` es `600` y de root (*"el secreto del servidor"*), y las dos filas de `curl`: sin `-k` falla, con `-k` anda.
**Qué decir:** *"el certificado autofirmado cifra igual que uno de verdad. Lo único que no tiene es a alguien que lo respalde. Por eso el navegador avisa, y por eso en producción lo firma la CA de la institución."*

---

## Firewall: hoy todo se abre dos veces

`sudo firewall-cmd --get-active-zones`.
**Qué señalar:** `internal` con `sources: 192.168.56.0/24`; `public` con las interfaces. En tu VM (UTM) las interfaces se llaman distinto (`enp0s1`, `enp0s2`); decirlo en voz alta.
**Qué decir:** *"todo lo que publiquemos hoy va en las dos zonas. Si desde `localhost:8080` carga y desde `192.168.56.10` no, es esto."*
