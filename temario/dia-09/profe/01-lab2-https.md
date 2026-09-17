# Lab — HTTPS con certificado propio (comandos)

## Parte 1 — Instalar el módulo
```bash
sudo dnf install -y mod_ssl
ls /etc/httpd/conf.d/
grep -n Listen /etc/httpd/conf.d/ssl.conf
grep -n localhost /etc/httpd/conf.d/ssl.conf
```

## Parte 2 — Reiniciar y ver el certificado
```bash
sudo apachectl configtest
sudo systemctl restart httpd
systemctl status httpd-init --no-pager | head -3
sudo ls -l /etc/pki/tls/certs/localhost.crt /etc/pki/tls/private/localhost.key
sudo openssl x509 -in /etc/pki/tls/certs/localhost.crt -noout -subject -issuer -dates
sudo ss -tlnp | grep httpd
```
Decir: "`httpd-init` corrió una vez, generó el certificado y se apagó: por eso dice `inactive (dead)`. `CN = rhel01` es el nombre de la VM. Emisor igual al sujeto: se firmó a sí mismo."

El texto exacto de `subject`/`issuer` depende de la versión: si sale distinto, lo que importa es que `issuer` no es ninguna CA conocida.

Si después del `restart` **no existe** `/etc/pki/tls/certs/localhost.crt`, generarlo a mano y seguir igual:
```bash
sudo openssl req -x509 -newkey rsa:2048 -nodes -days 365 -keyout /etc/pki/tls/private/localhost.key -out /etc/pki/tls/certs/localhost.crt -subj "/C=PA/O=PGN/CN=rhel01"
sudo chmod 600 /etc/pki/tls/private/localhost.key
sudo systemctl restart httpd
```

## Parte 3 — Abrir el 443 en las dos zonas
```bash
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --zone=internal --add-service=https --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
```
Decir: "`internal` trae de fábrica `mdns` y `samba-client`; por eso su lista es más larga".

## Parte 4 — Probar cifrado
```bash
curl https://localhost/
curl -k https://localhost/
curl -k -H "Host: intranet.lab.local" https://localhost/
```
Decir: "el tercero devuelve el sitio por defecto aunque pidamos `intranet`: el virtual host de la intranet existe solo en el 80. Para tenerla en HTTPS haría falta otro `<VirtualHost *:443>` con `SSLEngine on` y las dos rutas del certificado. No lo hacemos hoy."

## Parte 5 — Desde el navegador
En tu Mac: `https://<tu IP host-only>/`. Ellos: `https://192.168.56.10/`. Mostrar el aviso del navegador y "Continuar de todos modos".
