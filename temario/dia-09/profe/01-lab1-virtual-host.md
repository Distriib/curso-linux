# Lab — Dos sitios en una IP (comandos)

Yo hago el primero, ellos los demás, foto. En tu VM (UTM) la IP host-only puede no ser `192.168.56.10`: donde el material la nombra, usá la tuya; ellos usan `192.168.56.10`.

## Parte 1 — Cómo quedó Apache
```bash
sudo ss -tlnp | grep httpd
sudo apachectl -S
curl http://localhost/
sudo firewall-cmd --get-active-zones
```
Si a alguien le sale otro puerto o `403`: no hizo la tarea del Día 8.
```bash
sudo vim /etc/httpd/conf/httpd.conf
```
`/Listen 8` → `Listen 80`. `/DocumentRoot "` → `DocumentRoot "/var/www/html"`. `/Directory "` → `<Directory "/var/www/html">`. `:wq`.
```bash
sudo apachectl configtest
sudo systemctl restart httpd
curl http://localhost/
```
Si `restart` falla con `Address already in use`: hay dos líneas `Listen`. `grep -n Listen /etc/httpd/conf/httpd.conf` y borrar la sobrante con `dd`.

## Parte 2 — El sitio por defecto, con nombre
```bash
sudo vim /etc/httpd/conf.d/00-default.conf
```
```
<VirtualHost *:80>
    ServerName rhel01
    DocumentRoot /var/www/html
</VirtualHost>
```
```bash
sudo apachectl configtest
```
Decir: "el `00` en el nombre es a propósito: el primero que se carga es el default".

## Parte 3 — El contenido de la intranet
```bash
sudo mkdir -p /var/www/intranet/descargas
echo "<h1>Intranet PGN - rhel01</h1>" | sudo tee /var/www/intranet/index.html
echo "Documento 1 de la intranet" | sudo tee /var/www/intranet/descargas/doc1.txt
echo "Documento 2 de la intranet" | sudo tee /var/www/intranet/descargas/doc2.txt
echo "Documento 3 de la intranet" | sudo tee /var/www/intranet/descargas/doc3.txt
ls /var/www/intranet/descargas
ls -Z /var/www/intranet
```
Decir: "nació con `httpd_sys_content_t` sin que hiciéramos nada, porque está dentro de `/var/www`".

## Parte 4 — El virtual host
```bash
sudo vim /etc/httpd/conf.d/intranet.conf
```
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
Pegar el texto del archivo en el chat para los que se atrasen.

## Parte 5 — Misma IP, mismo puerto, distinto sitio
```bash
curl http://localhost/
curl -H "Host: intranet.lab.local" http://localhost/
```
Decir: "misma IP, mismo puerto, distinto contenido. Lo único que cambió es el nombre que pedimos".

## Parte 6 — Con nombre de verdad, y la lista de archivos
```bash
echo "127.0.0.1 intranet.lab.local intranet" | sudo tee -a /etc/hosts
curl http://intranet.lab.local/
curl -s http://intranet.lab.local/descargas/ | grep txt
curl http://intranet.lab.local/descargas/doc2.txt
curl http://intranet.lab.local/descargas/doc3.txt
```
Si alguien recibe la intranet al pedir `http://localhost/`: le falta `00-default.conf` o lo nombró de forma que ordena después de `intranet.conf`. `sudo apachectl -S` muestra el `default server`.

## Parte 7 — Cada sitio con su log
```bash
sudo tail -3 /var/log/httpd/intranet-access_log
```
