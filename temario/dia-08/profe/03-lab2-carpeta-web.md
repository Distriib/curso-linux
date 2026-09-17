# Lab — La web desde una carpeta propia: /web (comandos)

Vos primero, ellos después, foto.

## Parte 1 — La carpeta nueva y su etiqueta
```bash
sudo mkdir /web
echo "<h1>Portal institucional - servido desde /web</h1>" | sudo tee /web/index.html
ls -Zd /web
ls -Z /web
```
Si en alguna VM `/web` sale con otro tipo (por ejemplo `root_t`), `sudo restorecon -v /web` lo deja en `default_t` y el lab sigue igual.

## Parte 2 — Apuntar Apache a /web
```bash
sudo vim /etc/httpd/conf.d/web.conf
```
Contenido (`i`, escribir, `Esc`, `:wq`):
```
DocumentRoot "/web"
<Directory "/web">
    Require all granted
</Directory>
```
```bash
sudo apachectl configtest
sudo systemctl restart httpd
curl -I http://localhost:82
```
`DocumentRoot`: la carpeta desde la que Apache sirve las páginas (de fábrica `/var/www/html`, en `httpd.conf`). Como `httpd.conf` incluye al final todos los `conf.d/*.conf`, lo que pongamos ahí gana. El bloque `<Directory "/web"> Require all granted </Directory>` es el permiso de **Apache** para servir esa carpeta: sin él, Apache da 403 por su propia configuración, no por SELinux, y confunde el diagnóstico. Por eso van las cuatro líneas.
`apachectl configtest`: revisa la sintaxis de la configuración de Apache; `Syntax OK`. Puede imprimir además un aviso `AH00558 ... fully qualified domain name`: es ruido, no error.
Qué decir del 403: "configuración correcta, permisos correctos, y 403. Ya saben dónde mirar."

## Parte 3 — Confirmar con el AVC
```bash
sudo ausearch -m AVC -ts recent | tail -1
```
Puede salir sobre `/web` (`tclass=dir`) o sobre `/web/index.html` (`tclass=file`): en los dos casos `tcontext=...default_t` → fila 2 de la tabla.

## Parte 4 — La tentación: chcon
```bash
sudo chcon -R -t httpd_sys_content_t /web
curl -I http://localhost:82
sudo restorecon -Rv /web
curl -I http://localhost:82
```
Qué decir: "`chcon` arregló. `restorecon` lo deshizo. Eso es lo que le va a pasar a la web un día cualquiera si la dejan con `chcon`."

## Parte 5 — La solución: regla + aplicar
```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
sudo semanage fcontext -l -C
sudo restorecon -Rv /web
ls -Z /web
curl http://localhost:82
```
Las comillas dobles alrededor de `"/web(/.*)?"` son obligatorias: sin ellas la shell interpreta el `?` y el `(`. Si alguien recibe un error raro de sintaxis, es eso.
