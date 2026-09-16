# Día 09 — Servicios de red (HTTP, NFS + autofs, SMB, FTP) y contenedores con Podman
> Al terminar, el participante publica un sitio con virtual host y HTTPS en Apache, comparte archivos por NFS (montados a demanda con autofs), SMB y FTP, y despliega un contenedor rootless con Podman que arranca solo con el sistema mediante Quadlet.

**Ficha técnica cubierta:**
- RH134 M4 Servicios de Red: HTTP, NFS, SMB, FTP (módulo completo).
- RH134 M7 Contenedores: Introducción a Podman, Gestión de imágenes, Contenedores Linux (módulo completo).
- RH134 M6 Almacenamiento Avanzado: Auto montaje (autofs sobre NFS, mapas indirecto y directo).
- Refuerzo transversal de RH134 M5 (SELinux avanzado, Firewalld) aplicado a cada servicio y a los volúmenes de contenedores.

**Requisitos previos:**
- Snapshot `dia08-fin` tomado. VM `rhel01` encendida y acceso por `ssh -p 2222 student@localhost`.
- Suscripción activa: `sudo subscription-manager status` muestra *Content Access Mode is set to Simple Content Access* y `dnf repolist` lista BaseOS y AppStream. Hoy se descargan varios paquetes e imágenes: sin repos no hay clase.
- Adaptador host-only con IP estática `192.168.56.10/24` (Día 5). Verificar con `ip -4 addr show | grep 192.168`. En UTM la red host-only puede tener otro rango: donde el material dice `192.168.56.10`, cada participante usa **su** IP (`nmcli device show | grep IP4.ADDRESS`).
- `httpd` instalado y habilitado (Día 6 y Día 8). **Importante:** el Día 8 Apache estuvo en el puerto 82 (Lab 3.1) y en el 8082 (reto), con DocumentRoot en `/web` y `/sitio`; la **tarea del Día 8** lo devolvió a `Listen 80`, `DocumentRoot "/var/www/html"` y `<Directory "/var/www/html">`. El primer paso de hoy lo verifica y lo corrige si alguien no hizo la tarea, porque los contenedores usarán 8080, 8081, 8083 y 8085.
- **Firewall con dos zonas (Día 8):** la red host-only `192.168.56.0/24` quedó como origen de la zona `internal`; el NAT (port forwarding) entra por `public`, que es la zona por defecto. Regla de hoy: **todo lo que se publique se agrega en las dos zonas** (`--add-service=X` para `public` y `--zone=internal --add-service=X`), si no, "desde `localhost` carga pero desde `192.168.56.10` no". Verificar al empezar con `sudo firewall-cmd --get-active-zones`.
- Usuarios de PanamaTech creados el Día 3: `ana` (UID 2001) y `carlos` en el grupo `sistemas` (GID 3001), `pedro` (soporte), `laura` (auditoria), contraseña inicial `Pgn.2026`. Si falta `ana`, el Lab 3.1 la crea.
- `/datos` (LV `lv_datos` de `vg_datos`, Día 6) montado: hoy se usa como punto de montaje del mapa directo de autofs (`/datos/nfs`). Los exports NFS viven en `/srv/nfs` para no mezclar con los datos del LVM.
- Salida a Internet desde la VM (descarga de imágenes). Si la red del aula es lenta, el instructor distribuye `ubi9-httpd-24.tar` con `podman save/load` (ver Notas para el instructor).
- Conocimientos: `systemctl`, `firewall-cmd`, `semanage`/`restorecon`, `/etc/fstab` (Días 4, 6 y 8).

---

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:35 | 35 | Bloque 1 — HTTP | Repaso Día 8 (3 preguntas). Paso 0: verificar que Apache está en el 80 (tarea del Día 8). Estructura de httpd, virtual host por nombre con SELinux, índice de directorios, HTTPS con `mod_ssl` |
| 0:35–1:20 | 45 | Bloque 2 — NFS + autofs | Servidor NFS (`/etc/exports`, `exportfs`, firewall), cliente (`mount`, `fstab` con `_netdev,nofail`), autofs con mapa indirecto y directo |
| 1:20–1:45 | 25 | Bloque 3 — SMB | Samba: `smb.conf` mínimo, `samba_share_t`, `smbpasswd`, `smbclient`, `mount -t cifs`, acceso desde Windows/Mac |
| 1:45–2:00 | 15 | Descanso | |
| 2:00–2:10 | 10 | Bloque 3 (cont.) — FTP | Demo del instructor: `vsftpd` con usuarios locales enjaulados, booleano SELinux, prueba con `curl` |
| 2:10–3:30 | 80 | Bloque 4 — Podman | Conceptos, registros, `pull/run/ps/logs/exec`, volúmenes con `:Z`, firewall, Containerfile y `build`, skopeo, Quadlet + `loginctl enable-linger`, `podman generate systemd` (legado) |
| 3:30–3:50 | 20 | Reto individual | Ticket: contenedor `portal` con Quadlet + NFS solo lectura + autofs |
| 3:50–4:00 | 10 | Cierre | Resumen, cheatsheet, reboot de verificación (contenedores arriba sin sesión), snapshot `dia09-fin` |

---

## Prioridad si falta tiempo

**Imprescindible** (lo que el participante debe salir sabiendo y habiendo practicado):
- Apache: virtual host por nombre en `conf.d/`, `apachectl configtest`, DocumentRoot con contexto `httpd_sys_content_t`, servicio `http` en el firewall.
- NFS cliente + autofs: `mount -t nfs`, entrada de `fstab` con `_netdev,nofail`, mapa indirecto en `/etc/auto.master.d/` (objetivo RHCSA).
- Podman rootless: `podman pull`, `podman run -d --name -p`, `podman ps -a`, `logs`, `exec`, `stop/rm`, `images/rmi`.
- Volumen con `-v origen:destino:Z` y por qué sin `:Z` SELinux bloquea al contenedor.
- Contenedor como servicio con Quadlet (`~/.config/containers/systemd/*.container`) + `systemctl --user` + `loginctl enable-linger` (objetivo RHCSA).

**Importante:**
- HTTPS con `mod_ssl` y certificado autofirmado.
- Servidor NFS (`/etc/exports`, `exportfs -rav`, `showmount -e`) y mapa directo de autofs (`/-`).
- Samba completo (servidor + `smbclient` + `mount -t cifs` con archivo de credenciales).
- Containerfile mínimo + `podman build`, `podman tag`, `podman image prune`.
- Alternativa legada `podman generate systemd --new --files` (aceptada en el examen).

**Si sobra tiempo** (demo del instructor o tarea):
- FTP con `vsftpd` (demo).
- `skopeo inspect`, volúmenes con nombre (`podman volume`), `podman system df / reset`.
- Acceso a la carpeta Samba desde el Explorador de Windows / Finder.
- Demo entre dos VMs (NFS y SMB con el firewall realmente en medio).
- Mención de nginx y del sysctl `net.ipv4.ip_unprivileged_port_start`.

---

## Bloque 1 — HTTP: virtual hosts, índices y HTTPS (35 min)

### Conceptos (8 min)

**Repaso del Día 8 (3 preguntas, con la terminal abierta):**
1. ¿Con qué comando se permite a Apache escuchar en un puerto no estándar sin que SELinux lo bloquee? (`sudo semanage port -a -t http_port_t -p tcp 8082`; ver con `semanage port -l | grep http`).
2. ¿Qué diferencia hay entre `firewall-cmd --add-service=http` con y sin `--permanent`? (sin `--permanent` se pierde al recargar/reiniciar; con `--permanent` requiere `--reload` para aplicarse ahora). ¿Y en qué zona cae el tráfico que llega desde `192.168.56.0/24`? (en `internal`, por el `--add-source` del Día 8; el NAT cae en `public`: por eso hoy todo se abre en las dos zonas).
3. ¿Dónde se ve por qué SELinux bloqueó algo? (`sudo ausearch -m AVC -ts recent`, `sudo sealert -a /var/log/audit/audit.log`).

**Apache en RHEL 9 = paquete `httpd` (2.4).** Decir en clase:
- `/etc/httpd/conf/httpd.conf` es la "constitución": directivas globales (`Listen`, `DocumentRoot`, `ServerRoot`) y al final un `IncludeOptional conf.d/*.conf`.
- `/etc/httpd/conf.d/*.conf` son las "leyes": ahí van los sitios (virtual hosts) y los módulos con configuración propia (`ssl.conf`, `welcome.conf`). Se cargan en orden alfabético: eso importa.
- `/etc/httpd/conf.modules.d/*.conf` carga módulos (`LoadModule`). Rara vez se toca.
- `/var/www/html` es el DocumentRoot por defecto. `/var/log/httpd/access_log` y `error_log` son los logs; `/etc/httpd/logs` es un enlace a ese directorio.
- Antes de reiniciar, **siempre** `sudo apachectl configtest` (o `httpd -t`): un error de sintaxis deja el servicio caído.

**Virtual host por nombre.** Una sola IP y puerto sirven varios sitios: Apache mira la cabecera `Host:` de la petición y elige el `<VirtualHost>` cuyo `ServerName`/`ServerAlias` coincide. Analogía: una recepcionista en un edificio con una sola puerta que pregunta "¿a quién busca?" y lo dirige. Trampa clásica del examen: si ninguna coincide, Apache sirve **el primer VirtualHost definido**, no el DocumentRoot global. Por eso hoy creamos primero un `00-default.conf`.

**SELinux.** Todo lo que está bajo `/var/www` hereda `httpd_sys_content_t`. Si el DocumentRoot está en otra ruta (`/srv/web`, `/datos/web`), hay que declararlo: `semanage fcontext -a -t httpd_sys_content_t "/ruta(/.*)?"` y `restorecon -Rv`. Si el archivo se creó en `/root` y se movió con `mv`, arrastra el contexto viejo → 403 (lo vimos el Día 8).

**HTTPS.** `dnf install mod_ssl` agrega `ssl.conf` con `Listen 443` y, al reiniciar `httpd`, el servicio auxiliar `httpd-init` genera un certificado autofirmado en `/etc/pki/tls/certs/localhost.crt` (clave en `/etc/pki/tls/private/localhost.key`). Sirve para cifrar; no sirve para que un navegador confíe (avisará). En producción: certificado de la CA institucional o Let's Encrypt (requiere dominio público).

**nginx** existe en AppStream (`dnf install nginx`) y compite por el puerto 80: misma lógica de firewall y SELinux. **En el RHCSA** no se pide configurar Apache a fondo, pero sí se usa como vehículo: desplegar un `httpd` básico, publicarlo en el firewall y arreglar contextos/puertos de SELinux.

### Lab 1.1 — Apache en el 80, virtual host `intranet` e índice de directorios (17 min)

- **Objetivo:** confirmar que Apache está en el puerto 80 con su DocumentRoot estándar, publicar un segundo sitio por nombre con el contexto SELinux correcto y comprobar que el sitio por defecto sigue respondiendo.

1. Ver el estado que dejó la tarea del Día 8:
```bash
sudo grep -nE '^(Listen|DocumentRoot|<Directory "/)' /etc/httpd/conf/httpd.conf
sudo ss -tlnp | grep httpd
sudo semanage port -lC
sudo firewall-cmd --get-active-zones
```
Salida esperada (aprox.):
```
47:Listen 80
124:DocumentRoot "/var/www/html"
136:<Directory "/var/www/html">
LISTEN 0 511 *:80 *:* users:(("httpd",pid=1234,fd=4),...)
http_port_t                    tcp      82, 8082
internal
  sources: 192.168.56.0/24
public
  interfaces: enp0s3 enp0s8
```
Qué observar: `semanage port -lC` lista solo las personalizaciones locales; las etiquetas 82 y 8082 del Día 8 pueden quedarse (no molestan). Las dos zonas activas son la razón de abrir todo "dos veces" hoy. (En UTM las interfaces se llaman distinto: `nmcli device`.)

2. **Solo si el paso 1 no dio `Listen 80` / `/var/www/html`** (no se hizo la tarea del Día 8): corregir a mano. Dejar **una sola** línea `Listen 80` (con `vim`, no con `sed` a ciegas: dos líneas `Listen 80` duplicadas hacen fallar a Apache):
```bash
sudo vim /etc/httpd/conf/httpd.conf
```
Dentro de vim: `/^Listen` para buscar, dejar `Listen 80` y borrar (`dd`) cualquier otra `Listen`; `/^DocumentRoot` → `DocumentRoot "/var/www/html"`; `/^<Directory "/` → `<Directory "/var/www/html">`. Guardar con `:wq`. Verificar (esto lo hacen **todos**, también quienes no editaron nada):
```bash
sudo grep -c '^Listen' /etc/httpd/conf/httpd.conf
sudo grep -rn '^Listen' /etc/httpd/conf.d/ || echo "sin Listen extra en conf.d"
sudo apachectl configtest
sudo systemctl restart httpd
sudo ss -tlnp | grep ':80 '
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
```
Salida esperada:
```
1
sin Listen extra en conf.d
Syntax OK
LISTEN 0 511 *:80 *:* users:(("httpd",pid=2345,fd=4),...)
dhcpv6-client http ssh
dhcpv6-client http mdns samba-client ssh
```
Qué observar: el servicio `http` ya estaba en las dos zonas desde el Día 8. `internal` trae de fábrica `mdns` y `samba-client` (por eso su lista es más larga) y además tiene la rich rule de log de SSH del Día 8; si el grupo no hizo el bloque opcional de hardening del Día 8, en ambas zonas aparecerá también `cockpit`: nada de eso molesta. Si ya estuviera instalado `mod_ssl` de un día anterior, el `grep` de `conf.d/` devolverá `ssl.conf:Listen 443 https` en vez del mensaje: también es correcto. Si falta en alguna: `sudo firewall-cmd --add-service=http --permanent; sudo firewall-cmd --zone=internal --add-service=http --permanent; sudo firewall-cmd --reload`.

3. Recorrido por la estructura (lectura rápida, 1 min; los números de línea son orientativos):
```bash
ls /etc/httpd/
ls /etc/httpd/conf.d/
ls /etc/httpd/conf.modules.d/ | head -5
ls -ld /etc/httpd/logs /etc/httpd/modules
grep -En '^(ServerRoot|DocumentRoot|IncludeOptional|ErrorLog|CustomLog)' /etc/httpd/conf/httpd.conf
```
Salida esperada (aprox.):
```
conf  conf.d  conf.modules.d  logs  modules  run  state
autoindex.conf  README  userdir.conf  welcome.conf
00-base.conf 00-dav.conf 00-lua.conf 00-mpm.conf 00-optional.conf
lrwxrwxrwx. 1 root root 19 ... /etc/httpd/logs -> ../../var/log/httpd
lrwxrwxrwx. 1 root root 29 ... /etc/httpd/modules -> ../../usr/lib64/httpd/modules
31:ServerRoot "/etc/httpd"
124:DocumentRoot "/var/www/html"
182:ErrorLog "logs/error_log"
217:CustomLog "logs/access_log" combined
356:IncludeOptional conf.d/*.conf
```

4. Crear el virtual host por defecto (para que el sitio original siga siendo el "default"):
```bash
sudo tee /etc/httpd/conf.d/00-default.conf > /dev/null <<'EOF'
# Sitio por defecto: responde cuando ninguna cabecera Host coincide
<VirtualHost *:80>
    ServerName rhel01
    DocumentRoot /var/www/html
</VirtualHost>
EOF
```

5. Crear el contenido de la intranet (todos con los mismos archivos):
```bash
sudo mkdir -p /var/www/intranet/descargas
echo "<h1>Intranet PGN - rhel01</h1>" | sudo tee /var/www/intranet/index.html
for i in 1 2 3; do echo "Documento $i de la intranet" | sudo tee /var/www/intranet/descargas/doc$i.txt > /dev/null; done
ls -Z /var/www/intranet
matchpathcon /var/www/intranet
```
Salida esperada:
```
<h1>Intranet PGN - rhel01</h1>
unconfined_u:object_r:httpd_sys_content_t:s0 descargas  unconfined_u:object_r:httpd_sys_content_t:s0 index.html
/var/www/intranet	system_u:object_r:httpd_sys_content_t:s0
```
Qué observar: al crearse dentro de `/var/www`, heredan `httpd_sys_content_t`. `matchpathcon` dice qué contexto *debería* tener una ruta según la política (en RHEL 9 está marcado como obsoleto pero sigue funcionando; el equivalente moderno es `sudo semanage fcontext -l | grep '/var/www'`).

6. Crear el virtual host de la intranet:
```bash
sudo tee /etc/httpd/conf.d/intranet.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName intranet.lab.local
    ServerAlias intranet
    DocumentRoot /var/www/intranet
    ErrorLog logs/intranet-error_log
    CustomLog logs/intranet-access_log combined
    <Directory /var/www/intranet>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
EOF
sudo apachectl configtest
sudo systemctl reload httpd
```
Salida esperada: `Syntax OK`. Qué observar: `Options Indexes` habilita el listado automático de directorios sin `index.html`; `logs/` es relativo a `ServerRoot`, es decir `/var/log/httpd/`.

7. Probar con la cabecera Host y sin ella:
```bash
curl http://localhost/
curl -H "Host: intranet.lab.local" http://localhost/
```
Salida esperada:
```
<h1>Portal institucional - version 2</h1>     <- el index.html de /var/www/html que dejó el Día 8 (el texto puede variar)
<h1>Intranet PGN - rhel01</h1>
```
Qué observar: misma IP, mismo puerto, distinto contenido según `Host:`. `httpd -S` (o `apachectl -S`) lista los vhosts cargados y marca cuál es el `default server`. Comentar el `00-default.conf` y repetir el primer `curl` para ver la trampa (opcional, si hay tiempo).

8. Resolver el nombre en la propia VM y probar el índice de directorios:
```bash
echo "127.0.0.1 intranet.lab.local intranet" | sudo tee -a /etc/hosts
curl http://intranet.lab.local/
curl -s http://intranet.lab.local/descargas/ | grep -o 'doc[0-9].txt' | sort -u
curl -s http://intranet.lab.local/descargas/doc2.txt
```
Salida esperada:
```
<h1>Intranet PGN - rhel01</h1>
doc1.txt
doc2.txt
doc3.txt
Documento 2 de la intranet
```

9. Desde el equipo del participante (opcional, 2 min): agregar `192.168.56.10 intranet.lab.local` al archivo hosts (Windows: `C:\Windows\System32\drivers\etc\hosts` con Bloc de notas como administrador; macOS/Linux: `/etc/hosts` con sudo) y abrir `http://intranet.lab.local/` en el navegador. Alternativa sin tocar hosts: `http://localhost:8080/` (port forwarding 8080→80) muestra el sitio por defecto.

10. Ver el log propio del vhost:
```bash
sudo tail -3 /var/log/httpd/intranet-access_log
```
Salida esperada: líneas `127.0.0.1 - - [fecha] "GET /descargas/ HTTP/1.1" 200 ...`.

11. Repaso express del caso "DocumentRoot fuera de /var/www" (ya se hizo el Día 8 con `/web`; sale en el examen, así que se repite en 1 minuto con otra ruta; si el grupo va atrasado, queda como tarea):
```bash
sudo mkdir -p /srv/web/otro
ls -Zd /srv/web/otro
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
sudo restorecon -Rv /srv/web
ls -Zd /srv/web/otro
```
Salida esperada:
```
unconfined_u:object_r:var_t:s0 /srv/web/otro
Relabeled /srv/web from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /srv/web/otro from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
unconfined_u:object_r:httpd_sys_content_t:s0 /srv/web/otro
```
Qué observar: `var_t` no es legible por `httpd_t`. Sin la regla `semanage fcontext`, `restorecon` lo devolvería a `var_t`. Detalle curioso: `/srv/www` **no** serviría para esta demostración, porque la política ya trae una regla para `/srv/([^/]*/)?www(/.*)?` (`semanage fcontext -l | grep '/srv/'`); por eso hoy se usa `/srv/web`.

- **Checkpoint:** pegar en el chat la salida de
```bash
curl -s -H "Host: intranet.lab.local" http://localhost/ && sudo apachectl configtest
```

### Lab 1.2 — HTTPS con certificado autofirmado (10 min)

- **Objetivo:** habilitar HTTPS en Apache con `mod_ssl`, publicar 443 en el firewall y entender por qué el cliente desconfía del certificado.

1. Instalar `mod_ssl` y ver qué trae:
```bash
sudo dnf install -y mod_ssl
ls /etc/httpd/conf.d/
grep -En '^(Listen|SSLCertificateFile|SSLCertificateKeyFile|<VirtualHost)' /etc/httpd/conf.d/ssl.conf
```
Salida esperada:
```
00-default.conf  autoindex.conf  intranet.conf  README  ssl.conf  userdir.conf  welcome.conf
40:Listen 443 https
56:<VirtualHost _default_:443>
85:SSLCertificateFile /etc/pki/tls/certs/localhost.crt
92:SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
```

2. Reiniciar (esto dispara `httpd-init`, que crea el certificado si no existe) y comprobar:
```bash
sudo apachectl configtest
sudo systemctl restart httpd
systemctl status httpd-init --no-pager | head -3
sudo ls -l /etc/pki/tls/certs/localhost.crt /etc/pki/tls/private/localhost.key
sudo openssl x509 -in /etc/pki/tls/certs/localhost.crt -noout -subject -issuer -dates
sudo ss -tlnp | grep -E ':(80|443) '
```
Salida esperada (aprox.):
```
Syntax OK
● httpd-init.service - One-time temporary TLS key generation for httpd.service
     Loaded: loaded (/usr/lib/systemd/system/httpd-init.service; static)
     Active: inactive (dead) since ...
-rw-r--r--. 1 root root 1310 ... /etc/pki/tls/certs/localhost.crt
-rw-------. 1 root root 1704 ... /etc/pki/tls/private/localhost.key
subject=C = US, O = Unspecified, OU = ca-1234..., CN = rhel01
issuer=C = US, O = Unspecified, OU = ca-1234..., CN = rhel01
notBefore=...
notAfter=...
LISTEN 0 511 *:80  *:* users:(("httpd",...))
LISTEN 0 511 *:443 *:* users:(("httpd",...))
```
Qué observar: el `CN` es el hostname (`hostnamectl`), y el emisor (`issuer`) no es ninguna CA reconocida: es una CA temporal (`OU=ca-...`) que `sscg` (el generador que usa `httpd-init`) crea y descarta en la misma máquina, o directamente el mismo sujeto. A efectos prácticos, un certificado **autofirmado**: cifra, pero nadie externo lo respalda. ⚠️ Verificar en la VM antes de la clase el texto exacto de `subject`/`issuer` (depende de la versión de `sscg`) y que `httpd-init.service` exista en esa versión de `mod_ssl`. La clave privada es 600 y de root.

**Plan B (si `/etc/pki/tls/certs/localhost.crt` no existe tras el `restart`):** generar el certificado a mano y reiniciar. Sirve igual para el resto del lab.
```bash
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout /etc/pki/tls/private/localhost.key \
  -out /etc/pki/tls/certs/localhost.crt \
  -subj "/C=PA/O=PGN/CN=rhel01"
sudo chmod 600 /etc/pki/tls/private/localhost.key
sudo systemctl restart httpd
```

3. Firewall (en las dos zonas: NAT y host-only) y prueba:
```bash
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --zone=internal --add-service=https --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services; sudo firewall-cmd --zone=internal --list-services
curl https://localhost/
curl -k https://localhost/
curl -k -H "Host: intranet.lab.local" https://localhost/
```
Salida esperada:
```
success
success
success
dhcpv6-client http https ssh
dhcpv6-client http https mdns samba-client ssh
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html
...
<h1>Portal institucional - version 2</h1>
<h1>Portal institucional - version 2</h1>
```
Qué observar: el mensaje de `curl` puede ser `self-signed certificate` o `unable to get local issuer certificate` según cómo haya generado el certificado `sscg`; en ambos casos significa "no confío en quien lo firmó". `-k` ignora la validación del certificado (solo para pruebas). La tercera línea devuelve el sitio por defecto aunque pidamos `intranet`: el vhost de la intranet solo existe en `*:80`. Para tener la intranet en HTTPS haría falta un `<VirtualHost *:443>` propio con `SSLEngine on` y las dos directivas `SSLCertificate*` (dejarlo indicado; no es necesario hacerlo hoy):
```
<VirtualHost *:443>
    ServerName intranet.lab.local
    DocumentRoot /var/www/intranet
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/localhost.crt
    SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
</VirtualHost>
```

4. Desde el navegador del participante: `https://192.168.56.10/` → aviso de certificado no confiable → "Continuar de todos modos" → página. Explicar que ese aviso es exactamente lo que verá un usuario con un certificado autofirmado.

- **Checkpoint:**
```bash
curl -sk https://localhost/ | head -1; sudo firewall-cmd --list-services; sudo firewall-cmd --zone=internal --list-services
```

---

## Bloque 2 — NFS y autofs (45 min)

### Conceptos (8 min)

**NFS (Network File System)** es "el disco de red del mundo Unix": el servidor **exporta** directorios y el cliente los **monta** como si fueran locales. Decir en clase:
- RHEL 9 usa por defecto **NFSv4.2**: un solo puerto, `2049/tcp`. NFSv3 necesita además `rpcbind` (111) y `mountd` (20048); por eso el firewall tiene tres servicios: `nfs`, `rpc-bind`, `mountd`. Con solo `nfs` alcanza para clientes v4, pero `showmount -e` desde otra máquina necesita los otros dos.
- Identidad por **uid/gid numérico**: si `ana` es uid 2001 en el servidor y 2001 en el cliente, es la misma persona para NFS. En una institución eso se resuelve con un directorio central (IdM/LDAP/AD) y, si se quiere seguridad real, `sec=krb5`. Hoy: `sec=sys` (confiar en el uid del cliente).
- `root_squash` (por defecto): el root del cliente se convierte en `nobody` en el servidor. `no_root_squash` lo desactiva: cómodo en laboratorio, peligroso en producción.
- `sync`: el servidor confirma la escritura solo cuando llegó a disco (más seguro, algo más lento).
- `/etc/exports` es la lista de "quién puede entrar a qué sala y con qué llave". Formato: `directorio  cliente(opciones)`, **sin espacio** entre cliente y paréntesis.
- Cliente: tres formas de montar, de menos a más robusta: `mount` manual (se pierde al reiniciar), `/etc/fstab` con `_netdev,nofail` (permanente; `nofail` evita que el arranque se quede colgado si el servidor no responde), y **autofs**: monta a demanda al entrar al directorio y desmonta tras un tiempo sin uso. Evita montajes colgados y arranques lentos. **El RHCSA pide autofs.**
- autofs tiene dos tipos de mapa: **indirecto** (un directorio padre, p. ej. `/remoto`, con "claves" que se convierten en subdirectorios) y **directo** (`/-` como punto de montaje y rutas absolutas en el mapa). Los mapas se declaran en `/etc/auto.master` o, mejor, en archivos `*.autofs` dentro de `/etc/auto.master.d/`.
- Hoy la misma VM es servidor y cliente. Montar por NFS un export de la propia máquina (*loopback*) es perfectamente válido para el laboratorio y para practicar el examen, pero **no se hace en producción**: bajo presión de memoria el kernel puede bloquearse esperándose a sí mismo. **El firewall no filtra el tráfico local** (interfaz `lo`): la configuración de firewalld que hacemos es la que necesitaría un cliente real; el instructor lo demuestra desde su segunda VM.
- SELinux: los archivos montados por NFS se ven con tipo `nfs_t` en el cliente. Booleanos útiles: `use_nfs_home_dirs` (homes por NFS), `httpd_use_nfs` (Apache sirviendo contenido NFS), `samba_share_nfs`. En el servidor, `nfs_export_all_ro` y `nfs_export_all_rw` están activos por defecto (por eso exportar no requiere etiquetar).

### Lab 2.1 — Servidor NFS (12 min)

- **Objetivo:** exportar un directorio lectura/escritura a la red host-only y otro solo lectura a localhost, y publicar NFS en el firewall.

1. Instalar y habilitar:
```bash
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server
systemctl status nfs-server --no-pager | head -4
sudo ss -tlnp | grep -E ':(2049|111|20048) '
```
Salida esperada:
```
Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → ...
● nfs-server.service - NFS server and services
     Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; ...)
     Active: active (exited) since ...
LISTEN 0 4096 0.0.0.0:111   ... users:(("rpcbind",...))
LISTEN 0 4096 0.0.0.0:20048 ... users:(("rpc.mountd",...))
LISTEN 0 64   0.0.0.0:2049  ...
```
Qué observar: `active (exited)` es normal: nfsd vive en el kernel, no como proceso de usuario. El 2049 no muestra proceso por la misma razón.

2. Crear los directorios y su contenido inicial:
```bash
sudo mkdir -p /srv/nfs/compartido /srv/nfs/lectura
sudo chown student:student /srv/nfs/compartido
sudo chmod 755 /srv/nfs/compartido
echo "Solo lectura desde NFS - $(hostname)" | sudo tee /srv/nfs/lectura/README.txt
ls -ld /srv/nfs/*
```
Salida esperada:
```
Solo lectura desde NFS - rhel01
drwxr-xr-x. 2 student student 6 ... /srv/nfs/compartido
drwxr-xr-x. 2 root    root   24 ... /srv/nfs/lectura
```
Qué observar: `compartido` es de `student` (uid 1000) para que el mismo uid pueda escribir desde el cliente sin ser root.

3. Escribir `/etc/exports` (ajustar la red si la IP host-only no es 192.168.56.x):
```bash
sudo tee /etc/exports > /dev/null <<'EOF'
# Directorio           Cliente(opciones)            <- SIN espacio antes del paréntesis
/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)
/srv/nfs/lectura       127.0.0.1(ro,sync)
EOF
sudo exportfs -rav
sudo exportfs -v
```
Salida esperada:
```
exporting 192.168.56.0/24:/srv/nfs/compartido
exporting 127.0.0.1:/srv/nfs/lectura
/srv/nfs/compartido  192.168.56.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
/srv/nfs/lectura     127.0.0.1(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
```
Qué observar: `-r` re-exporta todo lo que dice el archivo, `-a` todas las entradas, `-v` detalla. `exportfs -v` muestra también los valores por defecto que no escribimos. En vez de `127.0.0.1` se puede escribir `localhost`.

4. Firewall (para clientes reales; en `internal` porque los clientes NFS llegarían por la red host-only, y en `public` por completar) y comprobación:
```bash
sudo firewall-cmd --add-service={nfs,rpc-bind,mountd} --permanent
sudo firewall-cmd --zone=internal --add-service={nfs,rpc-bind,mountd} --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services; sudo firewall-cmd --zone=internal --list-services
showmount -e localhost
```
Salida esperada:
```
success
success
success
dhcpv6-client http https mountd nfs rpc-bind ssh
dhcpv6-client http https mdns mountd nfs rpc-bind samba-client ssh
Export list for localhost:
/srv/nfs/lectura    127.0.0.1
/srv/nfs/compartido 192.168.56.0/24
```

5. Booleanos SELinux relacionados (solo mirar):
```bash
getsebool nfs_export_all_ro nfs_export_all_rw use_nfs_home_dirs
```
Salida esperada:
```
nfs_export_all_ro --> on
nfs_export_all_rw --> on
use_nfs_home_dirs --> off
```

- **Checkpoint:**
```bash
showmount -e localhost
```

### Lab 2.2 — Cliente NFS: montaje manual y `/etc/fstab` (10 min)

- **Objetivo:** montar los dos exports desde la misma VM, comprobar lectura/escritura y dejar una línea correcta en `fstab`.

1. Montar a mano:
```bash
sudo mkdir -p /mnt/nfs /mnt/lectura
sudo mount -t nfs 192.168.56.10:/srv/nfs/compartido /mnt/nfs
sudo mount -t nfs localhost:/srv/nfs/lectura /mnt/lectura
mount | grep nfs4
df -hT /mnt/nfs /mnt/lectura
```
Salida esperada (aprox.):
```
192.168.56.10:/srv/nfs/compartido on /mnt/nfs type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,...,proto=tcp,...,addr=192.168.56.10)
localhost:/srv/nfs/lectura on /mnt/lectura type nfs4 (rw,relatime,vers=4.2,...,addr=127.0.0.1)
Filesystem                         Type  Size  Used Avail Use% Mounted on
192.168.56.10:/srv/nfs/compartido  nfs4   17G  2.1G   15G  13% /mnt/nfs
localhost:/srv/nfs/lectura         nfs4   17G  2.1G   15G  13% /mnt/lectura
```
Qué observar: `vers=4.2` sin haberlo pedido. El `rw` de `/mnt/lectura` es la opción del *cliente*; el servidor lo exportó `ro` y manda.

2. Escribir, leer y probar la restricción:
```bash
echo "escrito desde el cliente $(date +%T)" > /mnt/nfs/prueba.txt
cat /mnt/nfs/prueba.txt
ls -l /srv/nfs/compartido/
touch /mnt/lectura/no-se-puede.txt
sudo touch /mnt/lectura/tampoco-root.txt
ls -Zd /mnt/nfs
```
Salida esperada:
```
escrito desde el cliente 10:42:07
-rw-r--r--. 1 student student 40 ... prueba.txt
touch: cannot touch '/mnt/lectura/no-se-puede.txt': Read-only file system
touch: cannot touch '/mnt/lectura/tampoco-root.txt': Read-only file system
system_u:object_r:nfs_t:s0 /mnt/nfs
```
Qué observar: el archivo aparece en el directorio del "servidor" con el mismo dueño porque el uid coincide. El contexto SELinux del cliente es `nfs_t`.

3. Desmontar y hacerlo permanente con `fstab`:
```bash
sudo umount /mnt/nfs /mnt/lectura
echo "192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/nfs
```
Salida esperada:
```
192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
Filesystem                         Size  Used Avail Use% Mounted on
192.168.56.10:/srv/nfs/compartido   17G  2.1G   15G  13% /mnt/nfs
```
Qué observar: `_netdev` = "espera a la red antes de montar"; `nofail` = "si no puedes, sigue arrancando". `daemon-reload` es necesario porque systemd convierte `fstab` en unidades `.mount` (las versiones más nuevas de `mount` avisan si se omite; la de RHEL 9 no siempre). `mount -a` antes de reiniciar es la red de seguridad: un error de sintaxis en `fstab` sin `nofail` deja la VM en modo emergencia.

4. Dejar la línea comentada: a partir de ahora el mecanismo persistente será autofs y no queremos dos métodos peleando por el mismo recurso:
```bash
sudo umount /mnt/nfs
sudo sed -i 's|^192.168.56.10:/srv/nfs/compartido|#&|' /etc/fstab
tail -1 /etc/fstab
sudo systemctl daemon-reload
```
Salida esperada: `#192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0`.

- **Checkpoint:**
```bash
tail -1 /etc/fstab; ls -l /srv/nfs/compartido/
```

### Lab 2.3 — autofs: mapa indirecto y mapa directo (15 min)

- **Objetivo:** montar los exports a demanda en `/remoto/compartido`, `/remoto/lectura` (indirecto) y `/datos/nfs` (directo), y observar el montaje y desmontaje automáticos.

1. Instalar y leer el mapa maestro:
```bash
sudo dnf install -y autofs
grep -Ev '^(#|$)' /etc/auto.master
```
Salida esperada:
```
/misc	/etc/auto.misc
/net	-hosts
+dir:/etc/auto.master.d
+auto.master
```
Qué observar: `+dir:/etc/auto.master.d` incluye cualquier `*.autofs` de ese directorio: ahí ponemos lo nuestro sin tocar el archivo original.

2. Crear el mapa maestro del laboratorio (indirecto con timeout corto + directo):
```bash
sudo tee /etc/auto.master.d/lab.autofs > /dev/null <<'EOF'
# punto-de-montaje   archivo-de-mapa       opciones
/remoto              /etc/auto.remoto      --timeout=60
/-                   /etc/auto.directo
EOF
```

3. Mapa indirecto: la primera columna es la **clave** (subdirectorio que aparecerá bajo `/remoto`):
```bash
sudo tee /etc/auto.remoto > /dev/null <<'EOF'
# clave        opciones     origen
compartido     -rw,sync     192.168.56.10:/srv/nfs/compartido
lectura        -ro          localhost:/srv/nfs/lectura
EOF
```

4. Mapa directo: la primera columna es la **ruta absoluta** completa:
```bash
sudo tee /etc/auto.directo > /dev/null <<'EOF'
/datos/nfs     -rw,sync     192.168.56.10:/srv/nfs/compartido
EOF
```

5. Arrancar autofs y observar lo que crea:
```bash
sudo systemctl enable --now autofs
systemctl is-active autofs
ls -ld /remoto /datos/nfs
ls /remoto
mount | grep autofs
```
Salida esperada:
```
active
drwxr-xr-x. 2 root root 0 ... /remoto
drwxr-xr-x. 2 root root 0 ... /datos/nfs
                                     <- /remoto aparece VACÍO: es normal
/etc/auto.remoto on /remoto type autofs (rw,relatime,fd=...,pgrp=...,timeout=60,minproto=5,maxproto=5,indirect,...)
/etc/auto.directo on /datos/nfs type autofs (rw,relatime,fd=...,pgrp=...,timeout=300,minproto=5,maxproto=5,direct,...)
```
Qué observar: autofs crea los directorios él mismo (no hay que hacer `mkdir`). En un mapa indirecto, `ls /remoto` no muestra las claves hasta que alguien las use: hay que escribir el nombre exacto. (Si se quiere verlas, `browse_mode = yes` en `/etc/autofs.conf`.)

6. Disparar los montajes:
```bash
ls /remoto/compartido
cat /remoto/lectura/README.txt
cat /datos/nfs/prueba.txt
mount | grep nfs4 | awk '{print $1, "->", $3}'
```
Salida esperada:
```
prueba.txt
Solo lectura desde NFS - rhel01
escrito desde el cliente 10:42:07
192.168.56.10:/srv/nfs/compartido -> /remoto/compartido
localhost:/srv/nfs/lectura -> /remoto/lectura
192.168.56.10:/srv/nfs/compartido -> /datos/nfs
```
Qué observar: nadie ejecutó `mount`; el simple acceso a la ruta lo hizo.

7. Ver el desmontaje por inactividad (60 s en `/remoto`; 300 s por defecto en el directo). Salir del directorio antes (`cd ~`), porque un shell "parado" dentro mantiene el montaje activo. autofs revisa los montajes cada `timeout/4` (15 s), así que el desmontaje ocurre entre 60 y 75 s después del último acceso:
```bash
cd ~
sleep 90; mount | grep nfs4 | awk '{print $3}'
```
Salida esperada: solo `/datos/nfs` (el directo aún no llegó a sus 300 s). Mientras corre el `sleep`, explicar `/etc/autofs.conf` (`timeout = 300`, `browse_mode`).

8. Herramientas de diagnóstico y ciclo de cambios:
```bash
sudo automount -m | head -20
sudo systemctl reload autofs
journalctl -u autofs --no-pager | tail -3
```
Salida esperada: `automount -m` lista cada punto de montaje con su mapa y entradas; `reload` relee el mapa maestro (obligatorio tras editar `*.autofs` o un mapa directo; los mapas indirectos se releen solos en el siguiente acceso, pero recargar nunca sobra).

- **Checkpoint:**
```bash
ls /remoto/compartido && mount | grep -E 'nfs4|autofs' | awk '{print $1, $3, $5}'
```

**Demo del instructor (si el tiempo lo permite):** desde su segunda VM (`rhel02`, IP host-only 192.168.56.11), `sudo mount -t nfs 192.168.56.10:/srv/nfs/compartido /mnt` funciona; luego en `rhel01` `sudo firewall-cmd --remove-service=nfs` (sin `--permanent`) y repetir el montaje desde `rhel02`: se queda colgado hasta el timeout. `Ctrl+C`, `sudo firewall-cmd --reload` en `rhel01` y vuelve a funcionar. Es la forma más clara de mostrar que el firewall sí importa cuando el cliente es otra máquina.

---

## Bloque 3 — SMB con Samba (25 min) y FTP (10 min)

### Conceptos (5 min)

**SMB/CIFS** es el protocolo de carpetas compartidas de Windows; **Samba** lo implementa en Linux. Decir en clase:
- Dos demonios: `smbd` (archivos e impresoras, `445/tcp`; el 139 es compatibilidad) y `nmbd` (nombres NetBIOS, `137-138/udp`, heredado; se suele habilitar por compatibilidad con equipos viejos). `winbind` integra con Active Directory: fuera del alcance de hoy.
- `security = user`: cada usuario Samba **debe existir como usuario Linux** y además tener contraseña Samba propia (`smbpasswd -a`), que puede ser distinta de la de Linux. Samba guarda su base en `/var/lib/samba/private/passdb.tdb` (`pdbedit -L` la lista).
- Permisos en dos capas: Samba (`valid users`, `writable`, `create mask`) **y** Linux (dueño, grupo, modo). Si una de las dos niega, se niega. Por eso hoy la carpeta es `root:sistemas 2775` (setgid para heredar el grupo, como en `/srv/sistemas` del Día 3).
- SELinux: la carpeta compartida necesita `samba_share_t` (o el booleano `samba_export_all_rw`, más permisivo). Para compartir los home: `samba_enable_home_dirs`.
- `testparm` valida `smb.conf` antes de reiniciar: es el `apachectl configtest` de Samba.
- Clientes: `smbclient` (interactivo tipo ftp), `mount -t cifs` (paquete `cifs-utils`) y el Explorador de Windows / Finder.

### Lab 3.1 — Servidor Samba y clientes (20 min)

- **Objetivo:** compartir `/srv/samba/compartido` con el grupo `sistemas`, autenticar a `ana` y acceder desde `smbclient`, `mount -t cifs` y, si se puede, desde Windows/Mac.

1. Paquetes, usuario y grupo (del Día 3; los comandos son idempotentes por si a alguien le falta algo):
```bash
sudo dnf install -y samba samba-client cifs-utils
sudo groupadd -f -g 3001 sistemas
id ana || sudo useradd -m -u 2001 -c "Ana Rodriguez - Sistemas" -G sistemas ana
echo 'Pgn.2026' | sudo passwd --stdin ana
sudo usermod -aG sistemas ana
id ana
```
Salida esperada: `uid=2001(ana) gid=2001(ana) groups=2001(ana),3001(sistemas)`. Qué observar: `groupadd -f` no falla si el grupo ya existe. Se vuelve a fijar la clave Linux de `ana` al valor del curso (`Pgn.2026`) porque FTP la usará después y porque el Día 3 pudo dejarla caducada (`chage`).

2. Carpeta, permisos y contexto SELinux:
```bash
sudo mkdir -p /srv/samba/compartido
sudo chown root:sistemas /srv/samba/compartido
sudo chmod 2775 /srv/samba/compartido
sudo semanage fcontext -a -t samba_share_t "/srv/samba(/.*)?"
sudo restorecon -Rv /srv/samba
ls -ldZ /srv/samba/compartido
```
Salida esperada:
```
Relabeled /srv/samba from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:samba_share_t:s0
Relabeled /srv/samba/compartido from ... to unconfined_u:object_r:samba_share_t:s0
drwxrwsr-x. 2 root sistemas unconfined_u:object_r:samba_share_t:s0 6 ... /srv/samba/compartido
```

3. `smb.conf` mínimo completo (respaldar el original, que es largo y sirve de referencia):
```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo tee /etc/samba/smb.conf > /dev/null <<'EOF'
[global]
    workgroup = PGN
    server string = Servidor de archivos rhel01
    security = user
    map to guest = Never
    log file = /var/log/samba/log.%m
    max log size = 50

[compartido]
    comment = Carpeta compartida del grupo sistemas
    path = /srv/samba/compartido
    valid users = @sistemas
    writable = yes
    browseable = yes
    create mask = 0664
    directory mask = 2775
EOF
testparm -s
```
Salida esperada:
```
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)
Server role: ROLE_STANDALONE

# Global parameters
[global]
	...
[compartido]
	comment = Carpeta compartida del grupo sistemas
	create mask = 0664
	directory mask = 02775
	path = /srv/samba/compartido
	read only = No
	valid users = @sistemas
```
Qué observar: `writable = yes` se muestra como `read only = No` (son sinónimos). `@sistemas` significa "cualquier miembro del grupo Linux sistemas".

4. Contraseña Samba, servicios y firewall:
```bash
sudo smbpasswd -a ana
sudo pdbedit -L
sudo systemctl enable --now smb nmb
systemctl is-active smb nmb
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --zone=internal --add-service=samba --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep -E ':(139|445) '
```
`smbpasswd` pide la clave dos veces: usar `Pgn.2026` (la clave Samba es independiente de la de Linux; hoy se usa la misma para no confundir). Alternativa sin interacción: `(echo Pgn.2026; echo Pgn.2026) | sudo smbpasswd -s -a ana`.
Salida esperada:
```
Added user ana.
ana:2001:
active
active
success
success
success
LISTEN 0 50 0.0.0.0:139 ... users:(("smbd",...))
LISTEN 0 50 0.0.0.0:445 ... users:(("smbd",...))
```

5. Cliente `smbclient` (interactivo):
```bash
smbclient -L localhost -U ana
smbclient //localhost/compartido -U ana
```
Dentro del prompt `smb: \>`:
```
ls
put /etc/hostname hostname.txt
ls
exit
```
Salida esperada (resumida; `-L` termina con `Reconnecting with SMB1 for workgroup listing` / `Unable to connect with SMB1 -- no workgroup available`: es normal, SMB1 está deshabilitado):
```
	Sharename       Type      Comment
	---------       ----      -------
	compartido      Disk      Carpeta compartida del grupo sistemas
	IPC$            IPC       IPC Service (Servidor de archivos rhel01)
...
putting file /etc/hostname as \hostname.txt (0.0 kb/s) (average 0.0 kb/s)
  .                                   D        0  ...
  ..                                  D        0  ...
  hostname.txt                        A        7  ...
```
Versión sin interacción para scripts: `smbclient //localhost/compartido -U ana%Pgn.2026 -c 'ls'`.

6. Verificar en el lado servidor que el archivo llegó con el dueño y el grupo correctos:
```bash
ls -l /srv/samba/compartido/
```
Salida esperada: `-rw-rw-r--. 1 ana sistemas 7 ... hostname.txt` (grupo heredado por el setgid, modo por `create mask`).

7. Montar con `cifs` (como haría un servidor Linux que consume la carpeta):
```bash
sudo mkdir -p /mnt/smb
sudo mount -t cifs //localhost/compartido /mnt/smb -o username=ana,uid=student,gid=sistemas
mount | grep cifs
touch /mnt/smb/desde-cifs.txt
ls -l /mnt/smb/ /srv/samba/compartido/
sudo umount /mnt/smb
```
Pide la clave Samba de `ana`. Salida esperada:
```
//localhost/compartido on /mnt/smb type cifs (rw,relatime,vers=3.1.1,cache=strict,username=ana,uid=1000,...,gid=3001,...)
/mnt/smb/:
-rw-rw-r--. 1 student sistemas 0 ... desde-cifs.txt
-rw-rw-r--. 1 student sistemas 7 ... hostname.txt
/srv/samba/compartido/:
-rw-rw-r--. 1 ana sistemas 0 ... desde-cifs.txt
-rw-rw-r--. 1 ana sistemas 7 ... hostname.txt
```
Qué observar: en el montaje los archivos "parecen" de `student` (opción `uid=`), pero el servidor los escribió como `ana`, que es quien se autenticó. `vers=3.1.1`: SMB moderno, nada de SMB1.

8. *(Si sobra tiempo; si no, queda como tarea y el instructor lo muestra.)* Montaje permanente con archivo de credenciales (nunca la clave en `fstab`):
```bash
sudo tee /etc/samba/cred-ana > /dev/null <<'EOF'
username=ana
password=Pgn.2026
EOF
sudo chmod 600 /etc/samba/cred-ana
echo "//localhost/compartido  /mnt/smb  cifs  credentials=/etc/samba/cred-ana,uid=student,gid=sistemas,_netdev,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -hT /mnt/smb
sudo umount /mnt/smb
sudo sed -i 's|^//localhost/compartido|#&|' /etc/fstab
sudo systemctl daemon-reload
```
Salida esperada: `df` muestra `//localhost/compartido cifs ...`. Comentamos la línea al final por la misma razón que con NFS: en la misma VM el orden de arranque (smb debe estar arriba antes de montar) puede confundir en el Día 10; en un cliente real la línea se deja activa.

9. Desde el equipo del participante (momento "wow", 2 min):
   - Windows: `Win+R` → `\\192.168.56.10\compartido` → usuario `ana`, clave `Pgn.2026`. Crear un archivo de texto y verlo aparecer con `ls -l /srv/samba/compartido/` en la VM.
   - macOS: Finder → `Cmd+K` → `smb://192.168.56.10/compartido`.
   - Si Windows recuerda credenciales viejas: `net use \\192.168.56.10 /delete` en un `cmd`.

10. *(Opcional, 30 s.)* Booleanos SELinux de Samba (solo mirar):
```bash
getsebool -a | grep -E '^samba_(export_all|enable_home|share_nfs)'
```
Salida esperada: `samba_enable_home_dirs --> off`, `samba_export_all_ro --> off`, `samba_export_all_rw --> off`, `samba_share_nfs --> off`.

- **Checkpoint:**
```bash
testparm -s 2>/dev/null | grep -A6 '^\[compartido\]'; ls -lZ /srv/samba/compartido/
```

### Lab 3.2 — FTP con vsftpd (10 min, demo del instructor; comandos para quien quiera repetirlo)

- **Objetivo:** ver un servidor FTP funcional con usuarios locales enjaulados, y entender por qué hoy se prefiere SFTP/HTTPS.

Decir antes de empezar: FTP viaja **en texto plano** (usuario, clave y datos). En 2026 se enseña porque sigue existiendo en infraestructura heredada (impresoras/escáneres que suben PDFs, PLCs, sistemas viejos que solo hablan FTP). Para transferencias nuevas: **SFTP** (ya lo tienen con `sshd`: `sftp -P 2222 student@localhost`) o HTTPS. `vsftpd` no está en el RHCSA.

1. Instalar y ver la configuración por defecto de RHEL 9:
```bash
sudo dnf install -y vsftpd
sudo cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.orig
grep -En '^(anonymous_enable|local_enable|write_enable|chroot_local_user|allow_writeable_chroot|listen|listen_ipv6)' /etc/vsftpd/vsftpd.conf
```
Salida esperada:
```
12:anonymous_enable=NO
15:local_enable=YES
18:write_enable=YES
114:listen=NO
123:listen_ipv6=YES
```
Qué observar: anónimo ya está deshabilitado; falta el enjaulado (`chroot`).

2. Agregar las claves del laboratorio:
```bash
sudo tee -a /etc/vsftpd/vsftpd.conf > /dev/null <<'EOF'

# --- Curso PGN: usuarios locales enjaulados en su home ---
chroot_local_user=YES
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=30100
EOF
sudo systemctl enable --now vsftpd
sudo firewall-cmd --add-service=ftp --permanent
sudo firewall-cmd --add-port=30000-30100/tcp --permanent
sudo firewall-cmd --zone=internal --add-service=ftp --permanent
sudo firewall-cmd --zone=internal --add-port=30000-30100/tcp --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep ':21 '
```
Salida esperada: `LISTEN 0 32 *:21 *:* users:(("vsftpd",...))`. Qué observar: `chroot_local_user=YES` encierra al usuario en su home; como el home es escribible, vsftpd exige `allow_writeable_chroot=YES` (si no, error 500 al entrar). El rango pasivo fijo permite abrirlo en el firewall.

3. SELinux: por defecto `ftpd_t` no puede leer ni escribir en los home:
```bash
getsebool -a | grep -E '^ftp'
sudo setsebool -P ftp_home_dir on
```
Salida esperada: lista con `ftp_home_dir --> off`, `ftpd_anon_write --> off`, `ftpd_full_access --> off`, ... y tras el `setsebool`, `getsebool ftp_home_dir` → `on`. Si en su versión de política no existe `ftp_home_dir`, usar `ftpd_full_access` (más permisivo). `ftpd_anon_write` solo aplica a subida anónima.

4. Probar con `curl` (listar, subir, listar):
```bash
curl ftp://localhost/ --user ana:Pgn.2026
curl -T /etc/hostname ftp://localhost/subido.txt --user ana:Pgn.2026
curl ftp://localhost/ --user ana:Pgn.2026
sudo ls -l /home/ana/
```
Salida esperada:
```
                                                    <- listado vacío la primera vez (los archivos ocultos del home no se listan)
-rw-r--r--    1 2001     2001            7 Sep 03 10:55 subido.txt
-rw-r--r--. 1 ana ana 7 ... subido.txt
```
Cliente interactivo alternativo: `sudo dnf install -y lftp && lftp -u ana,Pgn.2026 localhost` (`ls`, `bye`).

5. Si algo falla: `sudo journalctl -u vsftpd -n 20`, `sudo tail /var/log/xferlog`, `sudo ausearch -m AVC -ts recent | grep vsftpd`.

- **Checkpoint (solo para quien lo hizo):**
```bash
curl -s ftp://localhost/ --user ana:Pgn.2026 | wc -l
```

---

## Bloque 4 — Contenedores con Podman (80 min)

### Conceptos (10 min)

Vocabulario (con analogía de construcción):
- **Imagen**: plantilla inmutable, formada por **capas** de solo lectura (el plano de una casa: sistema base + software + configuración). Nombre completo: `registro/espacio/nombre:etiqueta`, p. ej. `registry.access.redhat.com/ubi9/httpd-24:latest`.
- **Contenedor**: una instancia en ejecución de una imagen (la casa construida). Es un proceso normal de Linux aislado con namespaces (PID, red, montajes, usuarios), limitado con cgroups y confinado por SELinux (`container_t`). Tiene una capa de escritura propia que desaparece al borrarlo: **lo que deba persistir va en un volumen**.
- **Registro**: catálogo de imágenes. `registry.access.redhat.com` (imágenes UBI, sin login), `registry.redhat.io` (catálogo completo de Red Hat, requiere `podman login` con la cuenta Developer), `docker.io`, `quay.io`. La lista de búsqueda está en `/etc/containers/registries.conf`.
- **UBI (Universal Base Image)**: RHEL redistribuible sin suscripción. `ubi9/ubi`, `ubi9/ubi-minimal`, `ubi9/httpd-24`, etc.

**Podman vs Docker**: misma CLI (`alias docker=podman` funciona), pero Podman **no tiene demonio**: cada contenedor es hijo del comando que lo lanzó (o de systemd), no de un servicio central que corre como root. Por eso es **rootless por diseño**: cada usuario tiene sus propias imágenes y contenedores en `~/.local/share/containers/`, con uid remapeados según `/etc/subuid` y `/etc/subgid`. Los de root viven en `/var/lib/containers/` y **no se ven** desde `student`. Herramientas hermanas: `skopeo` (inspecciona y copia imágenes sin descargarlas ni ejecutarlas) y `buildah` (construye imágenes; `podman build` lo usa por dentro).

**Límites de rootless que veremos hoy**: no puede publicar puertos del host menores de 1024 (sysctl `net.ipv4.ip_unprivileged_port_start`, por defecto 1024) y su red es de usuario (pasta/slirp4netns), no un bridge del sistema. Dentro del contenedor, en cambio, el proceso *sí* puede escuchar en 80: el límite es el puerto **del host**.

**SELinux y volúmenes**: un directorio del host montado con `-v` conserva su tipo (`user_home_t`, `var_t`...), que `container_t` no puede leer. `:Z` lo reetiqueta a `container_file_t` con una categoría MCS **privada** de ese contenedor; `:z` lo etiqueta como **compartido** entre contenedores. Dos contenedores con `:Z` sobre el mismo directorio se pisan la etiqueta: usar `:z`. Nunca `:Z` sobre `/home` completo o `/etc`.

**Contenedor como servicio**: un `podman run -d` muere con la sesión y no vuelve al reiniciar. La forma actual es **Quadlet**: un archivo `nombre.container` en `~/.config/containers/systemd/` que systemd (`--user`) convierte en `nombre.service`. Para que arranque sin que nadie inicie sesión: `loginctl enable-linger student`. La forma antigua, `podman generate systemd --new`, sigue instalada y se acepta en el examen RHCSA 9, pero está marcada como obsoleta.

### Lab 4.1 — Instalar, buscar, descargar y ejecutar (25 min)

- **Objetivo:** tener las herramientas, comprobar el modo rootless, descargar dos imágenes UBI y operar un contenedor web con los comandos básicos.

1. Instalar y comprobar el modo rootless:
```bash
sudo dnf install -y container-tools
podman --version
podman info --format 'arch={{.Host.Arch}} rootless={{.Host.Security.Rootless}} storage={{.Store.GraphRoot}}'
grep student /etc/subuid /etc/subgid
```
Salida esperada (aprox.):
```
podman version 5.4.0
arch=amd64 rootless=true storage=/home/student/.local/share/containers/storage
/etc/subuid:student:100000:65536
/etc/subgid:student:100000:65536
```
Qué observar: en la VM del instructor `arch=arm64`. Si `grep` no devuelve nada, el usuario no tiene rangos: `sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 student && podman system migrate`.

2. Registros configurados:
```bash
grep -Ev '^(#|$)' /etc/containers/registries.conf
ls /etc/containers/registries.conf.d/
```
Salida esperada:
```
unqualified-search-registries = ["registry.access.redhat.com", "registry.redhat.io", "docker.io"]
short-name-mode = "enforcing"
000-shortnames.conf  001-rhel-shortnames.conf  002-rhel-shortnames-overrides.conf
```
Qué observar: con `short-name-mode = "enforcing"`, `podman pull httpd` pregunta en cuál registro buscar. Regla del curso: **siempre nombres completos**.

3. Buscar y descargar:
```bash
podman search registry.access.redhat.com/ubi9 --limit 5
podman pull registry.access.redhat.com/ubi9/ubi
podman pull registry.access.redhat.com/ubi9/httpd-24
podman images
```
Salida esperada (resumida):
```
NAME                                        DESCRIPTION
registry.access.redhat.com/ubi9/ubi         Provides the latest release of the Red Hat Universal Base Image 9.
registry.access.redhat.com/ubi9/ubi-minimal ...
...
Trying to pull registry.access.redhat.com/ubi9/ubi:latest...
Getting image source signatures
Copying blob ...
Writing manifest to image destination
a1b2c3d4e5f6...
REPOSITORY                                   TAG     IMAGE ID      CREATED      SIZE
registry.access.redhat.com/ubi9/httpd-24     latest  9f8e7d6c5b4a  2 weeks ago  4xx MB
registry.access.redhat.com/ubi9/ubi          latest  a1b2c3d4e5f6  2 weeks ago  2xx MB
```
Qué observar: si la descarga es lenta, el instructor pasa el `.tar` (ver Notas) y se carga con `podman load -i ubi9-httpd-24.tar`.

4. Contenedor interactivo y efímero:
```bash
podman run -it --rm registry.access.redhat.com/ubi9/ubi bash
```
Dentro del contenedor (prompt `[root@abc123 /]#`):
```
head -2 /etc/os-release
ps aux
id
cat /etc/hostname
exit
```
(Si la imagen no trajera `ps`, `ls -d /proc/[0-9]*` muestra los mismos dos PIDs.)
Salida esperada:
```
NAME="Red Hat Enterprise Linux"
VERSION="9.x (Plow)"
USER  PID %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
root    1  0.0  0.0  ...  pts/0 Ss   ...   0:00 bash
root    8  0.0  0.0  ...  pts/0 R+   ...   0:00 ps aux
uid=0(root) gid=0(root) groups=0(root)
abc123def456
```
Qué observar: "root" dentro del contenedor es `student` fuera (remapeo de uid). PID 1 es `bash`: no hay systemd ni otros procesos. `--rm` lo borra al salir: `podman ps -a` no lo muestra.

5. Inspeccionar la imagen web antes de usarla (¿en qué puerto escucha? ¿como quién corre?):
```bash
podman image inspect registry.access.redhat.com/ubi9/httpd-24 --format 'puertos={{.Config.ExposedPorts}} usuario={{.Config.User}}'
```
Salida esperada: `puertos=map[8080/tcp:{} 8443/tcp:{}] usuario=1001`. Qué observar: la imagen ya está pensada para rootless: escucha en 8080, no en 80, y corre como uid 1001.

6. Lanzar el servidor web en segundo plano (comprobar antes que el 8080 esté libre):
```bash
sudo ss -tlnp | grep ':8080 ' || echo "8080 libre"
podman run -d --name web -p 8080:8080 registry.access.redhat.com/ubi9/httpd-24
podman ps
curl -sI http://localhost:8080/ | head -3
curl -s http://localhost:8080/ | grep -o '<title>.*</title>'
```
Salida esperada:
```
8080 libre
3c4d5e6f...
CONTAINER ID  IMAGE                                            COMMAND     CREATED        STATUS        PORTS                   NAMES
3c4d5e6f7a8b  registry.access.redhat.com/ubi9/httpd-24:latest  /usr/bin/run-http...  5 seconds ago  Up 5 seconds  0.0.0.0:8080->8080/tcp  web
HTTP/1.1 403 Forbidden
Date: ...
Server: Apache/2.4.x (Red Hat Enterprise Linux)
<title>Test Page for the HTTP Server on Red Hat Enterprise Linux</title>
```
Qué observar: `-p host:contenedor`. Sin contenido en `/var/www/html`, Apache responde su página de prueba (código 403 con HTML de bienvenida, igual que el `httpd` del host cuando no hay `index.html`): no es un error de permisos. ⚠️ Verificar en la VM antes de la clase: según la versión de la imagen puede responder `200` con una página propia de la imagen; el mensaje de clase ("todavía no hay contenido nuestro") no cambia.

7. Los cinco comandos del día a día:
```bash
podman logs web | tail -3
podman port web
podman top web
podman exec -it web bash
```
Dentro (prompt `bash-5.1$`):
```
id
ls -ld /var/www/html
httpd -v
exit
```
Y después:
```bash
podman inspect web --format 'estado={{.State.Status}} pid={{.State.Pid}} ip={{.NetworkSettings.IPAddress}}'
podman stats --no-stream web
```
Salida esperada (aprox.):
```
[Wed Sep 03 ...] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.x (Red Hat Enterprise Linux) ... configured -- resuming normal operations
8080/tcp -> 0.0.0.0:8080
USER  PID  PPID  %CPU  ELAPSED  TTY  TIME  COMMAND
1001  1    0     0.000 1m...    ?    0s    httpd -D FOREGROUND
...
uid=1001(default) gid=0(root) groups=0(root)
drwxrwxr-x. 2 default root 6 ... /var/www/html
Server version: Apache/2.4.x (Red Hat Enterprise Linux)
estado=running pid=12345 ip=
ID  NAME  CPU %  MEM USAGE / LIMIT  MEM %  NET IO  BLOCK IO  PIDS  CPU TIME  AVG CPU %
... web   0.00%  12MB / 3.8GB       ...
```
Qué observar: `ip=` vacío es normal en rootless (red de usuario, sin IP propia visible). `podman logs` muestra la salida estándar del proceso principal: es el `journalctl -u` de los contenedores.

8. Ciclo de vida:
```bash
podman stop web
podman ps -a --format '{{.Names}} {{.Status}}'
podman start web
podman ps --format '{{.Names}} {{.Status}}'
```
Salida esperada:
```
web
web Exited (0) 3 seconds ago
web
web Up 2 seconds
```

9. El límite de los puertos privilegiados (verlo fallar):
```bash
podman run -d --name web80 -p 80:8080 registry.access.redhat.com/ubi9/httpd-24
podman rm -f web80 2>/dev/null; sudo sysctl net.ipv4.ip_unprivileged_port_start
```
Salida esperada: un error de permiso al reservar el puerto 80 (el texto exacto depende de la versión: con podman 4 menciona `rootlessport cannot expose privileged port 80`; con pasta en podman 5, `bind: permission denied`) y luego `net.ipv4.ip_unprivileged_port_start = 1024`. Qué observar: además, el 80 ya lo usa el Apache del host. Se podría bajar el sysctl (`sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80` y persistirlo en `/etc/sysctl.d/`), pero la práctica habitual es publicar en >1024 y poner un proxy/firewall delante. **No lo cambiamos.**

- **Checkpoint:**
```bash
podman ps --format '{{.Names}} {{.Image}} {{.Ports}} {{.Status}}'
```

### Lab 4.2 — Volúmenes, SELinux (`:Z`) y acceso desde el host (15 min)

- **Objetivo:** servir contenido del host desde el contenedor, ver cómo SELinux lo bloquea sin `:Z` y publicar los puertos en el firewall para el navegador del participante.

1. Contenido en el host y primer intento **sin** `:Z`:
```bash
mkdir -p ~/web
echo "<h1>Hola desde el volumen ~/web en rhel01</h1>" > ~/web/index.html
ls -Z ~/web/index.html
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html registry.access.redhat.com/ubi9/httpd-24
curl -s http://localhost:8081/ | grep -o '<title>.*</title>'
podman logs web2 | tail -1
sudo ausearch -m AVC -ts recent | grep 'comm="httpd"' | tail -1
```
Salida esperada:
```
unconfined_u:object_r:user_home_t:s0 /home/student/web/index.html
<title>403 Forbidden</title>
[...] AH00035: access to /index.html denied (filesystem path '/var/www/html/index.html') because search permissions are missing on a component of the path
type=AVC msg=audit(...): avc:  denied  { read } for  pid=... comm="httpd" name="index.html" ... scontext=system_u:system_r:container_t:s0:c123,c456 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
```
Qué observar: permisos Linux correctos (644), pero `container_t` no puede leer `user_home_t` (el permiso denegado puede ser `read`, `getattr` o `search` sobre el directorio: lo importante es `tcontext=...user_home_t`). Este 403 sí es de permisos; el AVC lo confirma.

2. Corregir con `:Z`:
```bash
podman rm -f web2
podman run -d --name web2 -p 8081:8080 -v ~/web:/var/www/html:Z registry.access.redhat.com/ubi9/httpd-24
curl -s http://localhost:8081/
ls -Z ~/web/index.html
echo "<p>Actualizado a las $(date +%T)</p>" >> ~/web/index.html
curl -s http://localhost:8081/ | tail -1
```
Salida esperada:
```
<h1>Hola desde el volumen ~/web en rhel01</h1>
unconfined_u:object_r:container_file_t:s0:c123,c456 /home/student/web/index.html
<p>Actualizado a las 12:10:33</p>
```
Qué observar: la etiqueta cambió **en el host** (`container_file_t` con categorías MCS). El cambio del archivo se ve al instante: es un bind mount, no una copia.

3. Volumen con nombre (gestionado por Podman; para datos de aplicaciones, p. ej. bases de datos):
```bash
podman volume create datos
podman volume inspect datos --format '{{.Mountpoint}}'
podman volume ls
```
Salida esperada: `datos` y `/home/student/.local/share/containers/storage/volumes/datos/_data`. Qué observar: Podman etiqueta estos volúmenes solo; `:Z` no hace falta.

4. Firewall (dos zonas) y prueba desde el equipo del participante:
```bash
sudo firewall-cmd --add-port={8080,8081,8083}/tcp --permanent
sudo firewall-cmd --zone=internal --add-port={8080,8081,8083}/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports; sudo firewall-cmd --zone=internal --list-ports
```
Salida esperada: dos líneas `8080/tcp 8081/tcp 8083/tcp` (una por zona). El 8083 se abre ya porque lo usará la imagen propia del Lab 4.3 y se comprueba desde el navegador tras el reboot del Lab 4.4. En el navegador del participante: `http://192.168.56.10:8081/` → "Hola desde el volumen". Alternativa sin red host-only: agregar en VirtualBox una regla NAT `8081 → 8081` (se puede con la VM encendida) y abrir `http://localhost:8081/`. Qué observar: en rootless no hace falta `semanage port`: quien abre el puerto en el host es un proceso del usuario (`unconfined_t`), no un servicio confinado.

- **Checkpoint:**
```bash
curl -s http://localhost:8081/ | head -1; ls -Z ~/web/index.html
```

### Lab 4.3 — Construir una imagen: Containerfile, `build`, `tag`, skopeo y limpieza (10 min)

- **Objetivo:** crear una imagen propia a partir de UBI, ejecutarla y conocer las herramientas de gestión de imágenes.

1. Containerfile mínimo:
```bash
mkdir -p ~/miweb && cd ~/miweb
cat > Containerfile <<'EOF'
FROM registry.access.redhat.com/ubi9/ubi
RUN dnf -y install httpd && dnf clean all
RUN echo "<h1>Imagen miweb construida en rhel01</h1>" > /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
EOF
podman build -t miweb .
podman images | grep -E 'miweb|ubi'
```
Salida esperada (resumida; el `dnf install` tarda 1–2 min):
```
STEP 1/5: FROM registry.access.redhat.com/ubi9/ubi
STEP 2/5: RUN dnf -y install httpd && dnf clean all
...
Complete!
STEP 3/5: RUN echo ...
STEP 4/5: EXPOSE 80
STEP 5/5: CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
COMMIT miweb
Successfully tagged localhost/miweb:latest
localhost/miweb                             latest  ...  ...  3xx MB
registry.access.redhat.com/ubi9/httpd-24    latest  ...
registry.access.redhat.com/ubi9/ubi         latest  ...
```
Qué observar: cada instrucción es una capa; la imagen sin registro se guarda como `localhost/miweb`. `dnf` dentro de la construcción usa los repos UBI (y, en un host RHEL suscrito, también los del host).

2. Ejecutarla: dentro escucha en 80 (lo permite: es root *dentro* del namespace), fuera publicamos 8083:
```bash
podman run -d --name miweb -p 8083:80 miweb
curl -s http://localhost:8083/
cd ~
```
Salida esperada: `<h1>Imagen miweb construida en rhel01</h1>`.

3. Etiquetas, inspección remota y limpieza:
```bash
podman tag miweb miweb:1.0
podman images | grep miweb
skopeo inspect docker://registry.access.redhat.com/ubi9/ubi | head -12
podman image prune -f
podman system df
```
Salida esperada (resumida):
```
localhost/miweb  1.0     abcd1234  ...
localhost/miweb  latest  abcd1234  ...
{
    "Name": "registry.access.redhat.com/ubi9/ubi",
    "Digest": "sha256:...",
    "RepoTags": [ "9.0.0", "9.1", ..., "latest" ],
    "Created": "...",
    "Architecture": "amd64",
    ...
TYPE           TOTAL  ACTIVE  SIZE    RECLAIMABLE
Images         3      3       ...
Containers     4      3       ...
Local Volumes  1      0       ...
```
Qué observar: `tag` no copia nada: dos nombres, mismo IMAGE ID. `skopeo inspect` consulta el registro sin descargar la imagen (útil para ver etiquetas disponibles y arquitectura). `image prune` borra imágenes sin etiqueta ("dangling"); `podman rmi miweb:1.0` quitaría solo esa etiqueta; `podman rmi -f` fuerza aunque haya contenedores.

- **Checkpoint:**
```bash
podman images --format '{{.Repository}}:{{.Tag}}' | sort
```

### Lab 4.4 — El contenedor como servicio: Quadlet, linger y la forma legada (20 min)

- **Objetivo:** que el contenedor web arranque con el sistema, sin sesión abierta, gestionado por systemd.

1. Limpiar los contenedores manuales que usan el 8080 y el directorio `~/web` (recordar: dos `:Z` sobre la misma carpeta se pisan):
```bash
podman rm -f web web2
podman ps -a --format '{{.Names}} {{.Status}}'
```
Salida esperada: solo `miweb Up ...`.

2. Crear el archivo Quadlet:
```bash
mkdir -p ~/.config/containers/systemd
cat > ~/.config/containers/systemd/web.container <<'EOF'
[Unit]
Description=Servidor web intranet en contenedor (Quadlet)
After=network-online.target

[Container]
Image=registry.access.redhat.com/ubi9/httpd-24
PublishPort=8080:8080
Volume=/home/student/web:/var/www/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
EOF
/usr/libexec/podman/quadlet -dryrun -user 2>&1 | grep -E '^(ExecStart|Restart|WantedBy|SourcePath)'
```
Salida esperada:
```
SourcePath=/home/student/.config/containers/systemd/web.container
Restart=always
WantedBy=default.target
ExecStart=/usr/bin/podman run --name systemd-web --cidfile=%t/%N.cid --replace --rm --cgroups=split --sdnotify=conmon -d -v /home/student/web:/var/www/html:Z --publish 8080:8080 registry.access.redhat.com/ubi9/httpd-24
```
Qué observar: `-dryrun` muestra el `.service` que Quadlet generará; si el archivo tiene un error, lo dice aquí. El contenedor se llamará `systemd-web` (se puede fijar con `ContainerName=`). Detalle: en una unidad **de usuario** `network-online.target` no existe, así que systemd ignora ese `After=`; se deja escrito porque el mismo archivo, copiado a `/etc/containers/systemd/`, sirve tal cual como servicio del sistema, donde sí aplica.

3. Activar (nota: `enable` **no aplica** a unidades generadas; el `[Install]` ya las deja habilitadas):
```bash
systemctl --user daemon-reload
systemctl --user start web.service
systemctl --user enable web.service
systemctl --user status web.service --no-pager | head -6
podman ps --format '{{.Names}} {{.Ports}} {{.Status}}'
curl -s http://localhost:8080/ | head -1
```
Salida esperada:
```
Failed to enable unit: Unit /run/user/1000/systemd/generator/web.service is transient or generated.
● web.service - Servidor web intranet en contenedor (Quadlet)
     Loaded: loaded (/home/student/.config/containers/systemd/web.container; generated)
     Active: active (running) since ...
   Main PID: 23456 (conmon)
systemd-web 0.0.0.0:8080->8080/tcp Up 10 seconds
miweb 0.0.0.0:8083->80/tcp Up 8 minutes
<h1>Hola desde el volumen ~/web en rhel01</h1>
```
Qué observar: el mensaje de `enable` es esperado, no un fallo. Desde ahora el contenedor se maneja con `systemctl --user stop|start|restart web.service` y sus logs con `journalctl --user -u web.service`. Si se hace `podman stop systemd-web`, `Restart=always` lo vuelve a levantar.

4. Arranque sin sesión (linger):
```bash
loginctl show-user student | grep Linger
sudo loginctl enable-linger student
loginctl show-user student | grep Linger
ls /var/lib/systemd/linger/
```
Salida esperada:
```
Linger=no
Linger=yes
student
```
Qué observar: sin linger, `systemd --user` de `student` (y con él el contenedor) muere al cerrar la última sesión SSH y no arranca al reiniciar.

5. Forma legada (aceptada en el examen, marcada como obsoleta): generar la unidad a partir del contenedor `miweb` que ya corre. ⚠️ Verificar en la VM antes de la clase que `podman generate systemd --help` sigue existiendo en la versión instalada: el subcomando está deprecado desde Podman 4.4 y desaparecerá en algún momento; si ya no está, este paso se convierte en explicación oral y el material del examen es solo Quadlet.
```bash
cd ~
podman generate systemd --new --files --name miweb
mkdir -p ~/.config/systemd/user
mv container-miweb.service ~/.config/systemd/user/
grep ExecStart= ~/.config/systemd/user/container-miweb.service
podman rm -f miweb
systemctl --user daemon-reload
systemctl --user enable --now container-miweb.service
systemctl --user is-active container-miweb.service
podman ps --format '{{.Names}} {{.Ports}} {{.Status}}'
```
Salida esperada:
```
DEPRECATED command:
It is recommended to use Quadlets for running containers and pods under systemd.
...
/home/student/container-miweb.service
ExecStart=/usr/bin/podman run --cidfile=%t/%n.ctr-id --cgroups=no-conmon --rm --sdnotify=conmon --replace -d --name miweb -p 8083:80 miweb
Created symlink /home/student/.config/systemd/user/default.target.wants/container-miweb.service → ...
active
systemd-web 0.0.0.0:8080->8080/tcp Up 3 minutes
miweb 0.0.0.0:8083->80/tcp Up 5 seconds
```
Qué observar: `--new` genera una unidad que **crea** el contenedor con `podman run --replace` en cada arranque (por eso pudimos borrarlo antes); sin `--new`, la unidad solo hace `podman start` de un contenedor que debe existir. Aquí `enable --now` sí funciona porque es una unidad normal en `~/.config/systemd/user/`. Diferencias resumidas: Quadlet = archivo declarativo `.container`, sin `enable`; legado = `.service` generado, con `enable`.

6. La prueba real es reiniciar y comprobar **sin iniciar sesión**. Lo hacemos al final del reto (Cierre): `sudo reboot`, esperar 1 min y, desde el equipo propio, abrir `http://192.168.56.10:8080/` y `:8083/` **antes** de conectarse por SSH. Luego `ssh` y:
```bash
systemctl --user is-active web.service container-miweb.service
podman ps --format '{{.Names}} {{.Status}}'
```

7. Menciones finales (no ejecutar): `podman system reset` borra **todo** (imágenes, contenedores, volúmenes, configuración de almacenamiento) del usuario; `AutoUpdate=registry` en `[Container]` + `podman auto-update` actualiza la imagen y reinicia el servicio; `podman kube play` despliega YAML de Kubernetes en Podman (puente hacia OpenShift).

- **Checkpoint:**
```bash
systemctl --user is-active web.service container-miweb.service; loginctl show-user student | grep Linger
```

---

## Reto individual (20 min en clase; lo que falte se termina como tarea)

**Ticket #0931 — Portal institucional en contenedor**

> Solicitud de la Dirección de Informática: desplegar en `rhel01` un contenedor llamado `portal`, basado en la imagen `registry.access.redhat.com/ubi9/httpd-24`, que sirva el contenido del directorio `~/portal` del usuario `student` (debe existir un `index.html` con el texto "Portal PGN"). El portal debe responder en el puerto **8085** del host y ser accesible desde el navegador del participante (IP host-only o port forwarding). Debe arrancar solo al reiniciar la VM, **sin que nadie inicie sesión**. Además, el directorio `~/portal` debe exportarse por NFS a la red host-only en modo **solo lectura**, y montarse a demanda con autofs en `/remoto/portal`. Entregar: salida de `podman ps`, `curl http://localhost:8085/`, `showmount -e localhost` y `cat /remoto/portal/index.html`, más una captura del navegador.

Sin pistas. Criterio de aceptación: tras `sudo reboot`, `http://192.168.56.10:8085/` carga desde el equipo del participante antes de abrir SSH, y `cat /remoto/portal/index.html` funciona.

### Solución (para el instructor)

```bash
# 1. Contenido
mkdir -p ~/portal
echo "<h1>Portal PGN</h1>" > ~/portal/index.html

# 2. Quadlet
mkdir -p ~/.config/containers/systemd
cat > ~/.config/containers/systemd/portal.container <<'EOF'
[Unit]
Description=Portal PGN en contenedor
After=network-online.target

[Container]
Image=registry.access.redhat.com/ubi9/httpd-24
ContainerName=portal
PublishPort=8085:8080
Volume=/home/student/portal:/var/www/html:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user start portal.service
systemctl --user is-active portal.service
podman ps --format '{{.Names}} {{.Ports}} {{.Status}}'
curl -s http://localhost:8085/

# 3. Firewall y arranque sin sesión
sudo firewall-cmd --add-port=8085/tcp --permanent
sudo firewall-cmd --zone=internal --add-port=8085/tcp --permanent
sudo firewall-cmd --reload
sudo loginctl enable-linger student
# Quien no tenga red host-only: agregar en VirtualBox la regla NAT 8085 -> 8085
# (se puede con la VM encendida) y probar en http://localhost:8085/

# 4. NFS solo lectura hacia la red host-only
#    /home/student es 700 por defecto: el servidor NFS necesita al menos "search"
#    para que un cliente pueda recorrer la ruta hasta el export.
sudo chmod 711 /home/student
echo "/home/student/portal   192.168.56.0/24(ro,sync)" | sudo tee -a /etc/exports
sudo exportfs -rav
showmount -e localhost

# 5. autofs (mapa indirecto ya existente /remoto -> /etc/auto.remoto)
echo "portal   -ro   192.168.56.10:/home/student/portal" | sudo tee -a /etc/auto.remoto
sudo systemctl reload autofs
cat /remoto/portal/index.html
mount | grep portal

# 6. Verificación final
sudo reboot
# desde el equipo propio, sin SSH: http://192.168.56.10:8085/  -> "Portal PGN"
# luego: ssh -p 2222 student@localhost
systemctl --user is-active portal.service; podman ps --format '{{.Names}} {{.Status}}'; cat /remoto/portal/index.html
```

Errores típicos del reto: olvidar `loginctl enable-linger` (funciona hasta el reboot); `PublishPort=8085:80` en vez de `:8080` (la imagen escucha en 8080); volumen sin `:Z` (403); abrir 8085 solo en `public` y probar desde la IP host-only (zona `internal`); `/etc/exports` con espacio antes del paréntesis (exporta a todo el mundo con opciones por defecto: `exportfs -v` lo delata); no recargar autofs tras editar el mapa; si `ls /remoto/portal` da *Permission denied*, comprobar que `/home/student` permita al menos búsqueda (`chmod 711 /home/student`) y que la exportación sea de `/home/student/portal`, no de `~/portal` literal.

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `sudo apachectl configtest` / `httpd -t` | Validar la configuración de Apache antes de reiniciar |
| `curl -H "Host: intranet.lab.local" http://localhost/` | Probar un virtual host por nombre sin tocar DNS |
| `sudo semanage fcontext -a -t httpd_sys_content_t "/ruta(/.*)?"` + `restorecon -Rv` | Contexto SELinux para un DocumentRoot fuera de `/var/www` |
| `sudo dnf install mod_ssl` → `curl -k https://localhost/` | HTTPS con certificado autofirmado (`-k` ignora la validación) |
| `sudo openssl x509 -in /etc/pki/tls/certs/localhost.crt -noout -subject -dates` | Ver a quién pertenece y cuándo vence un certificado |
| `sudo exportfs -rav` / `sudo exportfs -v` | Aplicar `/etc/exports` y ver qué está exportado con qué opciones |
| `showmount -e servidor` | Listar los exports NFS de un servidor |
| `sudo mount -t nfs servidor:/export /mnt/x` | Montar NFS a mano |
| `servidor:/export /mnt/x nfs defaults,_netdev,nofail 0 0` | Línea de `fstab` para NFS |
| `/etc/auto.master.d/*.autofs` + `/etc/auto.mapa` | Declarar mapas autofs (indirecto: `/dir mapa`; directo: `/- mapa`) |
| `sudo systemctl reload autofs` / `sudo automount -m` | Recargar mapas / volcar la configuración de autofs |
| `testparm -s` | Validar `smb.conf` |
| `sudo smbpasswd -a usuario` / `sudo pdbedit -L` | Crear/listar usuarios Samba |
| `smbclient -L servidor -U usuario` / `smbclient //srv/share -U usuario` | Listar recursos / cliente interactivo SMB |
| `sudo mount -t cifs //srv/share /mnt/x -o credentials=/etc/samba/cred,uid=...` | Montar SMB con archivo de credenciales (600) |
| `curl ftp://host/ --user u:p` / `curl -T archivo ftp://host/ --user u:p` | Listar / subir por FTP |
| `sudo setsebool -P ftp_home_dir on` | Permitir a vsftpd acceder a los home (booleano) |
| `sudo firewall-cmd --add-service={nfs,rpc-bind,mountd,samba,ftp,https} --permanent` (+ lo mismo con `--zone=internal`) + `--reload` | Publicar servicios en el firewall, en la zona del NAT y en la de la red host-only |
| `sudo firewall-cmd --get-active-zones` | Ver qué zona atiende cada interfaz/origen (`public` NAT, `internal` host-only) |
| `httpd -S` | Listar los virtual hosts cargados y cuál es el default |
| `podman pull registro/espacio/imagen` / `podman images` / `podman rmi` | Descargar, listar y borrar imágenes |
| `podman run -d --name n -p host:cont -v /dir:/dest:Z imagen` | Ejecutar un contenedor con puerto y volumen etiquetado |
| `podman ps -a` / `logs` / `exec -it n bash` / `inspect` / `port` / `top` | Operación diaria de contenedores |
| `podman stop` / `start` / `rm -f` | Ciclo de vida |
| `podman build -t nombre .` / `podman tag` / `podman image prune` | Construir, etiquetar, limpiar |
| `skopeo inspect docker://registro/imagen` | Inspeccionar una imagen remota sin descargarla |
| `podman save -o f.tar imagen` / `podman load -i f.tar` | Exportar/importar imágenes sin red |
| `~/.config/containers/systemd/n.container` → `systemctl --user daemon-reload && systemctl --user start n` | Quadlet: contenedor como servicio de usuario |
| `/usr/libexec/podman/quadlet -dryrun -user` | Depurar un archivo Quadlet |
| `sudo loginctl enable-linger usuario` | Que los servicios de usuario arranquen sin sesión |
| `podman generate systemd --new --files --name n` | Forma legada de generar la unidad systemd |
| `sudo ausearch -m AVC -ts recent` | Ver bloqueos de SELinux (contenedores, httpd, samba, ftp) |

---

## Notas para el instructor

### Preparar antes de la clase
- Restaurar `dia08-fin` en una VM de pruebas y ejecutar **todos** los labs de corrido, cronometrando; el día es denso y el orden importa (Apache al 80 antes de Podman; `web`/`web2` borrados antes del Quadlet).
- Confirmar en esa VM el estado que deja la tarea del Día 8 (`Listen 80`, `DocumentRoot "/var/www/html"`, `<Directory "/var/www/html">`, zona `internal` con origen `192.168.56.0/24` y `http` en ambas zonas). Quien no hizo la tarea tendrá un 403 o un puerto equivocado: el paso 2 del Lab 1.1 lo arregla.
- ⚠️ Verificar en la VM antes de la clase el `subject`/`issuer` exacto del certificado que genera `httpd-init` (`sscg`) y el mensaje de `curl https://localhost/`, para no contradecir la salida esperada del Lab 1.2.
- Pre-descargar imágenes y generar el paquete para repartir si la red del aula es lenta:
```bash
podman pull registry.access.redhat.com/ubi9/ubi registry.access.redhat.com/ubi9/httpd-24
podman save -o ubi9-httpd-24.tar registry.access.redhat.com/ubi9/httpd-24
podman save -o ubi9-ubi.tar registry.access.redhat.com/ubi9/ubi
# repartir por chat/USB; cada participante: scp -P 2222 ubi9-*.tar student@localhost:/tmp/ ; podman load -i /tmp/ubi9-httpd-24.tar
```
  Ojo: las imágenes son multi-arch; un `.tar` guardado desde el Mac aarch64 contiene la variante **arm64** y no sirve en VirtualBox x86_64. Generar los `.tar` desde una VM x86_64 (o pedir a un participante que lo haga la noche anterior) o, mejor, que cada uno haga el `pull` como tarea del Día 8.
- ⚠️ Verificar en la VM antes de la clase: que `podman generate systemd --help` sigue disponible (Lab 4.4 paso 5), que la imagen `ubi9/httpd-24` responde con la página de prueba (403 + HTML) al arrancar sin contenido, y que `podman build` con el Containerfile del Lab 4.3 levanta el contenedor (si falla por `/run/httpd`, agregar `RUN mkdir -p /run/httpd`).
- Tener la segunda VM `rhel02` (192.168.56.11) lista para las demos NFS/SMB con el firewall en medio.
- Probar el acceso `\\192.168.56.10\compartido` desde un Windows real: es el momento "wow" y conviene que salga a la primera.
- Tener a mano el texto del `smb.conf`, `intranet.conf`, `web.container` y `Containerfile` para pegarlos en el chat (los participantes que se atrasen copian y pegan).
- Verificar que `student` tiene `subuid/subgid` en la VM de referencia; si el usuario se creó en el instalador sí los tiene.
- Snapshot `dia09-inicio` antes de empezar por si hay que restaurar a alguien.

### Qué estudiar si es nuevo en RHEL (la noche anterior, en la VM)
1. **NFS + autofs**: `man 5 exports`, `man 5 auto.master`, `man 5 autofs`. Practicar: exportar, montar desde localhost, mapa indirecto y directo, romperlo (quitar el reload, poner espacio en exports) y arreglarlo. Es objetivo RHCSA.
2. **Podman rootless y Quadlet**: `man podman-run`, `man podman-systemd.unit` (sección Quadlet), `man podman-generate-systemd`. Practicar el flujo completo: `run -d -p -v :Z` → `.container` → `daemon-reload` → `start` → `enable-linger` → `reboot` → comprobar sin sesión. Entender por qué `systemctl --user enable` falla con Quadlet.
3. **SELinux con contenedores**: `man container_selinux`; reproducir el 403 sin `:Z`, leer el AVC con `ausearch`, entender `:Z` vs `:z`. Con su perfil de seguridad, esto le resultará natural y es el punto que más "vende" SELinux.
4. **Samba**: `man smb.conf` (secciones `valid users`, `create mask`, `security`), `testparm`. Practicar login desde Windows y el error clásico `NT_STATUS_ACCESS_DENIED` (SELinux o permisos Linux).
5. **Apache virtual hosts**: `httpd -S` (lista los vhosts y cuál es el default), orden alfabético de `conf.d/`, `httpd-init` y `mod_ssl`.

### Errores frecuentes de los participantes y cómo resolverlos

| Síntoma | Causa | Solución |
|---|---|---|
| `httpd` no arranca tras editar `Listen`: `Address already in use` | Dos líneas `Listen 80` (o `sed` que duplicó) | `grep -n '^Listen' /etc/httpd/conf/httpd.conf`, dejar una; `apachectl configtest` |
| `curl http://localhost/` devuelve la intranet en vez del sitio original | No existe `00-default.conf` o se llama de forma que ordena después de `intranet.conf` | Crear `00-default.conf` con `ServerName rhel01`; `httpd -S` muestra el default |
| 403 en `/srv/web/...` con permisos correctos | Contexto `var_t` | `semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"` + `restorecon -Rv` |
| `curl http://localhost/` da 403 en el sitio por defecto nada más empezar | La tarea del Día 8 no se hizo: `<Directory "/sitio">` o `DocumentRoot "/sitio"` siguen en `httpd.conf` | Paso 2 del Lab 1.1: restaurar `DocumentRoot` y `<Directory "/var/www/html">`, `configtest`, `restart` |
| "Desde `localhost:8080` (NAT) carga, desde `192.168.56.10` no" (https, 8081, 8085, Samba) | El servicio/puerto se abrió solo en `public`; el origen host-only cae en `internal` | `firewall-cmd --zone=internal --add-service/--add-port ... --permanent` + `--reload`; `--get-active-zones` |
| `exportfs: ... does not support NFS export` o export a `*` | Espacio entre cliente y `(opciones)` en `/etc/exports` | Quitar el espacio; `exportfs -rav`; `exportfs -v` |
| `mount.nfs: access denied by server` | La IP origen no está en el export (UTM con otra red host-only) o `exportfs -r` no se ejecutó | Ajustar la red en `/etc/exports`; `exportfs -rav`; `showmount -e` |
| `ls /remoto` vacío "no funciona" | Comportamiento normal del mapa indirecto | `ls /remoto/compartido` (nombre exacto) o `browse_mode = yes` |
| autofs no toma el cambio del mapa | Falta `systemctl reload autofs` o typo en `*.autofs` | `sudo automount -m`; `journalctl -u autofs` |
| No desmonta tras el timeout | Un shell está dentro del directorio, o aún no pasó el ciclo de expiración (hasta timeout + timeout/4) | `cd ~` y esperar; `sudo fuser -vm /remoto/compartido` muestra quién lo usa |
| `smbclient`: `NT_STATUS_LOGON_FAILURE` | Falta `smbpasswd -a ana` o clave distinta de la Linux | `sudo smbpasswd -a ana`; `pdbedit -L` |
| `smbclient`: `NT_STATUS_ACCESS_DENIED` al escribir | Contexto no es `samba_share_t`, o `ana` no está en `sistemas`, o falta el setgid/2775 | `ls -ldZ`, `restorecon -Rv /srv/samba`; `id ana`; `chmod 2775` |
| Windows no conecta a `\\192.168.56.10\compartido` | Firewall sin `samba`; credenciales cacheadas; VM sin IP host-only | `firewall-cmd --list-services`; `net use \\ip /delete`; `ip -4 addr` |
| vsftpd: `500 OOPS: cannot change directory:/home/ana` | Booleano SELinux `ftp_home_dir` off | `setsebool -P ftp_home_dir on` |
| vsftpd: `500 OOPS: vsftpd: refusing to run with writable root inside chroot()` | Falta `allow_writeable_chroot=YES` | Agregarla y `systemctl restart vsftpd` |
| `podman pull httpd` pregunta el registro o falla | Nombre corto con `short-name-mode=enforcing` | Usar nombre completo `registry.access.redhat.com/ubi9/httpd-24` |
| `unauthorized: authentication required` al hacer pull | Imagen de `registry.redhat.io` sin login | `podman login registry.redhat.io` o usar `registry.access.redhat.com` |
| `podman ps` no muestra el contenedor que "acabo de crear" | Se creó con `sudo podman` (root) y se consulta como `student` (o al revés) | Usar siempre el mismo usuario; `sudo podman ps` para ver los de root |
| `port is already allocated` / `address already in use` en 8080 | Apache del Día 8 sigue en 8080, o el contenedor `web` no se borró antes del Quadlet | `sudo ss -tlnp \| grep 8080`; volver Apache al 80; `podman rm -f web` |
| `cannot expose privileged port 80` | Rootless no publica <1024 | Usar `-p 8080:8080`; mención de `ip_unprivileged_port_start` |
| 403 desde el contenedor con volumen | Falta `:Z` | `podman rm -f` y volver a lanzar con `:Z`; `ausearch -m AVC` |
| Tras el Quadlet, `web2` (mismo directorio) da 403 | Dos `:Z` sobre `~/web` se pisaron la etiqueta | Borrar `web2` antes, o usar `:z` en ambos |
| `systemctl --user`: `Failed to connect to bus` | Se hizo `sudo -i` y luego `su - student` (sin sesión de systemd) | Salir y entrar por SSH directamente como `student` |
| `Failed to enable unit: ... is transient or generated` | Es Quadlet: no se hace `enable` | Solo `daemon-reload` + `start`; el `[Install]` lo habilita |
| Quadlet no genera el servicio (`Unit web.service not found`) | Typo en la sección/clave o archivo fuera de `~/.config/containers/systemd/` | `/usr/libexec/podman/quadlet -dryrun -user` |
| El contenedor no está tras el reboot | Falta `loginctl enable-linger student` | Habilitar linger; comprobar `ls /var/lib/systemd/linger/` |
| `newuidmap: write to uid_map failed` / `cannot setup namespace` | Usuario sin `subuid/subgid` | `sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 student; podman system migrate` |
| `miweb` arranca y muere; `podman logs miweb` muestra `could not create /run/httpd/httpd.pid` | El directorio `/run/httpd` no quedó en la imagen | Agregar `RUN mkdir -p /run/httpd` antes del `CMD` en el Containerfile y reconstruir con `podman build -t miweb .` |
| `podman build` falla en `dnf install` | Sin red o repos UBI inaccesibles | Verificar `curl -I https://cdn-ubi.redhat.com`; reintentar; usar `ubi9/httpd-24` ya descargada |

### Diferencias VirtualBox (x86_64) vs UTM (aarch64)
- Las imágenes UBI son multi-arch: el mismo `podman pull` trae `amd64` en VirtualBox y `arm64` en UTM. `podman info --format '{{.Host.Arch}}'` y `skopeo inspect ... | grep Architecture` lo muestran. Un `.tar` de `podman save` **no** es intercambiable entre arquitecturas.
- La red host-only en UTM ("Host Only") puede no ser `192.168.56.0/24`: el instructor debe decir en voz alta su IP y recordar que los participantes usan la suya en `exports`, `auto.remoto` y el navegador.
- En UTM, el port forwarding NAT existe solo con "Emulated VLAN"; para las pruebas desde el navegador del Mac conviene usar directamente la IP host-only.
- Nombres de interfaz: `enp0s3/enp0s8` (VirtualBox) vs `enp0s1/enp0s2` aprox. (UTM): verificar con `nmcli device`. Hoy no se toca la red, solo se lee la IP.
- El Explorador de Windows (`\\ip\compartido`) vs Finder (`smb://ip/compartido`): mismo servidor, misma cuenta `ana`.

### Preguntas probables y respuesta corta
- **¿Podman reemplaza a Docker? ¿Puedo usar mis Dockerfile y docker-compose?** Sí: `Containerfile` y `Dockerfile` son el mismo formato; la CLI es compatible. Para compose, `podman-compose` (EPEL) o `podman compose` como envoltorio; en RHEL la ruta recomendada es Quadlet/`podman kube play`.
- **¿Por qué rootless si al final el servicio es "del sistema"?** Un contenedor comprometido en rootless es un proceso de `student`, no de root: menos daño posible. Con linger, systemd de usuario lo mantiene vivo igual que un servicio del sistema.
- **¿NFS es seguro?** Con `sec=sys` confía en el uid del cliente: solo para redes controladas. Para más seguridad, `sec=krb5p` con IdM/Kerberos, y nunca `no_root_squash` en producción.
- **¿autofs o fstab?** fstab para montajes que deben estar siempre (datos de una aplicación); autofs para muchos clientes/recursos que se usan a ratos (homes, repositorios de documentos) y para evitar arranques colgados.
- **¿Se puede integrar Samba con nuestro Active Directory?** Sí (`realmd`/`sssd`/`winbind`, `security = ads`); es material de RH294/RH362, no de este curso.
- **¿Por qué FTP si es inseguro?** Porque existe en equipos que no hablan otra cosa. Para todo lo nuevo: SFTP (ya tienen `sshd`) o HTTPS. `vsftpd` soporta TLS (`ssl_enable=YES`) si no hay alternativa.
- **¿Qué pasa con mis contenedores `podman run -d` cuando reinicio?** No vuelven. Por eso existe Quadlet (o `--restart=always` con `podman-restart.service`, menos recomendable).
- **¿HTTPS "de verdad" con Let's Encrypt?** Requiere un dominio público que apunte al servidor; en una intranet se usa la CA institucional y se distribuye su certificado raíz a los clientes.
- **¿Cuánto pesa una imagen? ¿Dónde se guarda?** `podman system df` y `podman image inspect --format '{{.Size}}'`; en `~/.local/share/containers/storage` (rootless) o `/var/lib/containers/storage` (root).

### Relación con el examen RHCSA (EX200, RHEL 9)
- **Manage containers** (objetivo completo del examen): buscar y descargar imágenes de un registro remoto; inspeccionar imágenes (`podman inspect`, `skopeo inspect`); gestionar contenedores con `podman` y `skopeo`; ejecutar, iniciar, detener y listar contenedores; ejecutar un servicio dentro de un contenedor; **configurar un contenedor para arrancar automáticamente como servicio systemd** (Quadlet o `generate systemd`, ambos aceptados); **adjuntar almacenamiento persistente** (`-v` con `:Z`).
- **Create and configure file systems**: montar y desmontar sistemas de archivos de red con NFS; **configurar autofs**.
- **Manage security**: configurar firewalld; restaurar contextos por defecto; **gestionar etiquetas de puertos SELinux** (Día 8) y **booleanos** (`ftp_home_dir`, `use_nfs_home_dirs`); diagnosticar violaciones de política (`ausearch`).
- Apache aparece en el examen como **vehículo** de tareas de SELinux/firewall (DocumentRoot no estándar, puerto no estándar), no como servicio a configurar a fondo. El servidor NFS, Samba y FTP **no** son objetivos del RHCSA 9 (eran del RHCE 6/7); se incluyen porque están en la ficha del cliente y en su infraestructura real.

---

## Tarea y preparación para el día siguiente

1. **Snapshot `dia09-fin`** con la VM apagada, después de verificar el reboot (contenedores `systemd-web`, `miweb` y `portal` arriba sin sesión; `ls /remoto/compartido` funciona). Si algo quedó roto, restaurar `dia08-fin` y rehacer solo el Bloque 4.
2. Terminar el **Reto #0931** si no se completó en clase y pegar en el chat del curso las cuatro salidas pedidas.
3. **Práctica de 20 min** (memoria muscular para el examen), desde cero y sin mirar el material:
   - Un contenedor con `ubi9/httpd-24` en el puerto 8090 con un volumen `:Z`, convertido a servicio con Quadlet, y comprobar tras `systemctl --user restart`.
   - Un mapa autofs indirecto nuevo (`/nfs/lectura` → `localhost:/srv/nfs/lectura`) y comprobar el montaje a demanda.
   - Un virtual host `pruebas.lab.local` con DocumentRoot en `/srv/web/pruebas` con el contexto SELinux correcto (y abrirlo en las dos zonas del firewall si se prueba desde el navegador).
4. Lecturas cortas: `man podman-systemd.unit` (sección EXAMPLES), `man 5 auto.master` (sección "Format of the master map").
5. **El Día 10 es Troubleshooting y proyecto final**: el instructor romperá la VM de cada participante (servicio, firewall, permisos, disco, SELinux, y ahora también un contenedor y un montaje NFS). Repasar el método del Día 8: *servicio → logs → red/firewall → permisos → disco → SELinux*, y tener a mano los cheatsheets de los Días 6, 8 y 9.
6. Verificar que la VM tiene espacio: `df -h /` (las imágenes ocupan ~1 GB); si queda menos de 3 GB, `podman image prune -f` y `sudo dnf clean all`.
