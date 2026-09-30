# Lab 1.1 — Dos sitios en una IP

Vamos a confirmar que Apache quedó en el puerto 80, y a publicar un segundo sitio, `intranet`, en la misma IP y el mismo puerto.

| Sitio | Nombre | Carpeta | Archivo |
|---|---|---|---|
| por defecto | `rhel01` | `/var/www/html` | `conf.d/00-default.conf` |
| intranet | `intranet.lab.local` (alias `intranet`) | `/var/www/intranet` | `conf.d/intranet.conf` |

---

## Parte 1 — Cómo quedó Apache

**¿En qué puerto escucha Apache y qué carpeta sirve?**

```bash
sudo ss -tlnp | grep httpd
sudo apachectl -S
curl http://localhost/
sudo firewall-cmd --get-active-zones
```

**Comprobar:**
```
LISTEN 0 511 *:80 *:* users:(("httpd",pid=...
Main DocumentRoot: "/var/www/html"
<h1>Portal institucional - version 2</h1>
internal
  sources: 192.168.56.0/24
public
  interfaces: enp0s3 enp0s8
```
El texto del `<h1>` es el que dejó el Día 8; puede variar.

Si en vez de `*:80` sale otro puerto, o el `curl` da `403`, la tarea del Día 8 quedó sin hacer:
```bash
sudo vim /etc/httpd/conf/httpd.conf
```
Buscar `/Listen 8` y dejar `Listen 80`. Buscar `/DocumentRoot "` y dejar `DocumentRoot "/var/www/html"`. Buscar `/Directory "` y dejar `<Directory "/var/www/html">`. Guardar (`:wq`).
```bash
sudo apachectl configtest
sudo systemctl restart httpd
curl http://localhost/
```

---

## Parte 2 — El sitio por defecto, con nombre

**¿Cómo fijo cuál es el sitio que responde cuando nadie pide un nombre?**

```bash
sudo vim /etc/httpd/conf.d/00-default.conf
```
Pegar (`i`, pegar, `Esc`, `:wq`):
```
<VirtualHost *:80>
    ServerName rhel01
    DocumentRoot /var/www/html
</VirtualHost>
```

**Comprobar:**
```bash
sudo apachectl configtest
```
```
Syntax OK
```

---

## Parte 3 — El contenido de la intranet

**¿Qué etiqueta SELinux recibe una carpeta nueva dentro de `/var/www`?**

```bash
sudo mkdir -p /var/www/intranet/descargas
echo "<h1>Intranet PGN - rhel01</h1>" | sudo tee /var/www/intranet/index.html
echo "Documento 1 de la intranet" | sudo tee /var/www/intranet/descargas/doc1.txt
```

Hacer lo mismo con `doc2.txt` y `doc3.txt`, con `Documento 2` y `Documento 3` adentro.

**Comprobar:**
```bash
ls /var/www/intranet/descargas
ls -Z /var/www/intranet
```
```
doc1.txt  doc2.txt  doc3.txt
unconfined_u:object_r:httpd_sys_content_t:s0 descargas  unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```

---

## Parte 4 — El virtual host

**¿Cómo le digo a Apache que `intranet.lab.local` es otra carpeta?**

```bash
sudo vim /etc/httpd/conf.d/intranet.conf
```
Pegar:
```
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
```
```bash
sudo apachectl configtest
sudo systemctl reload httpd
sudo apachectl -S
```

**Comprobar:**
```
Syntax OK
*:80                   is a NameVirtualHost
         default server rhel01 (/etc/httpd/conf.d/00-default.conf:1)
         port 80 namevhost rhel01 (/etc/httpd/conf.d/00-default.conf:1)
         port 80 namevhost intranet.lab.local (/etc/httpd/conf.d/intranet.conf:1)
                 alias intranet
```

---

## Parte 5 — Misma IP, mismo puerto, distinto sitio

**¿Cómo pido un sitio por nombre sin tener DNS?**

```bash
curl http://localhost/
curl -H "Host: intranet.lab.local" http://localhost/
```

**Comprobar:**
```
<h1>Portal institucional - version 2</h1>
<h1>Intranet PGN - rhel01</h1>
```

---

## Parte 6 — Con nombre de verdad, y la lista de archivos

**¿Cómo hago que `intranet.lab.local` resuelva en mi propia VM?**

```bash
echo "127.0.0.1 intranet.lab.local intranet" | sudo tee -a /etc/hosts
curl http://intranet.lab.local/
curl -s http://intranet.lab.local/descargas/ | grep txt
curl http://intranet.lab.local/descargas/doc2.txt
```

Repetir el último `curl` pidiendo `doc3.txt`.

**Comprobar:**
```
<h1>Intranet PGN - rhel01</h1>
... <a href="doc1.txt">doc1.txt</a> ...
... <a href="doc2.txt">doc2.txt</a> ...
... <a href="doc3.txt">doc3.txt</a> ...
Documento 2 de la intranet
```
La lista de archivos la arma Apache solo, por `Options Indexes`.

---

## Parte 7 — Cada sitio con su log

**¿Dónde quedan registrados los pedidos a la intranet?**

```bash
sudo tail -3 /var/log/httpd/intranet-access_log
```

**Comprobar:**
```
127.0.0.1 - - [...] "GET /descargas/ HTTP/1.1" 200 ...
127.0.0.1 - - [...] "GET /descargas/doc2.txt HTTP/1.1" 200 ...
```

---

# Solución — todos los comandos

```bash
# Parte 1 — cómo quedó Apache
sudo ss -tlnp | grep httpd
sudo apachectl -S
curl http://localhost/
sudo firewall-cmd --get-active-zones

# Parte 2 — el sitio por defecto, con nombre
sudo vim /etc/httpd/conf.d/00-default.conf
```
Contenido (`i`, pegar, `Esc`, `:wq`):
```
<VirtualHost *:80>
    ServerName rhel01
    DocumentRoot /var/www/html
</VirtualHost>
```
```bash
sudo apachectl configtest

# Parte 3 — el contenido de la intranet
sudo mkdir -p /var/www/intranet/descargas
echo "<h1>Intranet PGN - rhel01</h1>" | sudo tee /var/www/intranet/index.html
echo "Documento 1 de la intranet" | sudo tee /var/www/intranet/descargas/doc1.txt
echo "Documento 2 de la intranet" | sudo tee /var/www/intranet/descargas/doc2.txt
echo "Documento 3 de la intranet" | sudo tee /var/www/intranet/descargas/doc3.txt
ls /var/www/intranet/descargas
ls -Z /var/www/intranet

# Parte 4 — el virtual host
sudo vim /etc/httpd/conf.d/intranet.conf
```
Contenido:
```
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
```
```bash
sudo apachectl configtest
sudo systemctl reload httpd
sudo apachectl -S

# Parte 5 — misma IP, mismo puerto, distinto sitio
curl http://localhost/
curl -H "Host: intranet.lab.local" http://localhost/

# Parte 6 — con nombre de verdad
echo "127.0.0.1 intranet.lab.local intranet" | sudo tee -a /etc/hosts
curl http://intranet.lab.local/
curl -s http://intranet.lab.local/descargas/ | grep txt
curl http://intranet.lab.local/descargas/doc2.txt
curl http://intranet.lab.local/descargas/doc3.txt

# Parte 7 — cada sitio con su log
sudo tail -3 /var/log/httpd/intranet-access_log
```

**Cómo elige Apache el sitio.** Dos sitios comparten la misma IP y el mismo puerto 80. Lo que los distingue es la cabecera `Host:` que manda el navegador. Por eso `curl http://localhost/` da el portal y `curl -H "Host: intranet.lab.local" http://localhost/` da la intranet: es el mismo servidor, el mismo puerto, y solo cambió el nombre pedido.

**Por qué hace falta `00-default.conf`.** Si hay virtual hosts y ninguno coincide con el nombre pedido, Apache responde con **el primero que leyó** — y los archivos de `conf.d/` se leen en orden alfabético. El `00-` adelante garantiza que el sitio por defecto sea el portal y no la intranet.

**No hizo falta tocar SELinux** en todo el lab: `/var/www/intranet` está dentro de `/var/www`, que la política ya tiene en su tabla como `httpd_sys_content_t`. Una carpeta fuera de ahí (como el `/web` del Día 8) sí habría necesitado `semanage fcontext`.

**`/etc/hosts` en vez de DNS:** la línea que se agrega hace que `intranet.lab.local` resuelva a `127.0.0.1` solo en esta VM. En producción ese nombre lo publicaría el servidor DNS.
