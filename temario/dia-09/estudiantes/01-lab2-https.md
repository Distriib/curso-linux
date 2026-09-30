# Lab 1.2 — HTTPS con certificado propio

Vamos a hacer que Apache también escuche en el 443, cifrado, con un certificado generado en la propia VM.

---

## Parte 1 — Instalar el módulo

**¿Qué agrega `mod_ssl` a la configuración?**

```bash
sudo dnf install -y mod_ssl
ls /etc/httpd/conf.d/
grep -n Listen /etc/httpd/conf.d/ssl.conf
grep -n localhost /etc/httpd/conf.d/ssl.conf
```

**Comprobar:**
```
00-default.conf  autoindex.conf  intranet.conf  README  ssl.conf  userdir.conf  welcome.conf
40:Listen 443 https
85:SSLCertificateFile /etc/pki/tls/certs/localhost.crt
92:SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
```
Los números de línea pueden variar.

---

## Parte 2 — Reiniciar y ver el certificado

**¿Quién generó el certificado y a nombre de quién está?**

```bash
sudo apachectl configtest
sudo systemctl restart httpd
systemctl status httpd-init --no-pager | head -3
sudo ls -l /etc/pki/tls/certs/localhost.crt /etc/pki/tls/private/localhost.key
sudo openssl x509 -in /etc/pki/tls/certs/localhost.crt -noout -subject -issuer -dates
sudo ss -tlnp | grep httpd
```

**Comprobar:**
```
Syntax OK
● httpd-init.service - One-time temporary TLS key generation for httpd.service
     Active: inactive (dead) since ...
-rw-r--r--. 1 root root ... /etc/pki/tls/certs/localhost.crt
-rw-------. 1 root root ... /etc/pki/tls/private/localhost.key
subject=C = US, O = Unspecified, OU = ca-..., CN = rhel01
issuer=C = US, O = Unspecified, OU = ca-..., CN = rhel01
notBefore=...
notAfter=...
LISTEN 0 511 *:80  ...
LISTEN 0 511 *:443 ...
```
`CN = rhel01`: el nombre de la VM. `issuer` igual que `subject`: se firmó a sí mismo. La clave privada es `600`.

---

## Parte 3 — Abrir el 443 en las dos zonas

**¿Por qué hay que abrir el puerto dos veces?**

```bash
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --zone=internal --add-service=https --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
```

**Comprobar:**
```
success
success
success
dhcpv6-client http https ssh
dhcpv6-client http https mdns samba-client ssh
```
Puede aparecer también `cockpit`: no molesta.

---

## Parte 4 — Probar cifrado

**¿Por qué `curl` se niega, si el sitio funciona?**

```bash
curl https://localhost/
curl -k https://localhost/
curl -k -H "Host: intranet.lab.local" https://localhost/
```

**Comprobar:**
```
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html
...
<h1>Portal institucional - version 2</h1>
<h1>Portal institucional - version 2</h1>
```
El mensaje puede decir `unable to get local issuer certificate`: significa lo mismo, "no confío en quien lo firmó". La tercera línea devuelve el sitio por defecto aunque pidamos `intranet`: el virtual host de la intranet existe solo en el puerto 80.

---

## Parte 5 — Desde el navegador

**¿Qué ve un usuario cuando el certificado es autofirmado?**

En el navegador de tu computadora: `https://192.168.56.10/`.

**Comprobar:** el navegador avisa que la conexión no es privada o que el certificado no es confiable. Con "Continuar de todos modos" carga la página. Ese aviso es lo que ve cualquier usuario con un certificado autofirmado.

---

# Solución — todos los comandos

```bash
# Parte 1 — instalar mod_ssl
sudo dnf install -y mod_ssl
ls /etc/httpd/conf.d/
grep -n Listen /etc/httpd/conf.d/ssl.conf
grep -n localhost /etc/httpd/conf.d/ssl.conf

# Parte 2 — reiniciar y mirar el certificado
sudo apachectl configtest
sudo systemctl restart httpd
systemctl status httpd-init --no-pager | head -3
sudo ls -l /etc/pki/tls/certs/localhost.crt /etc/pki/tls/private/localhost.key
sudo openssl x509 -in /etc/pki/tls/certs/localhost.crt -noout -subject -issuer -dates
sudo ss -tlnp | grep httpd

# Parte 3 — abrir el 443 en las DOS zonas
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --zone=internal --add-service=https --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services

# Parte 4 — probar cifrado
curl https://localhost/            # falla: certificado autofirmado
curl -k https://localhost/         # -k = no verifiques el certificado
curl -k -H "Host: intranet.lab.local" https://localhost/

# Parte 5 — desde el navegador de tu computadora:
#   https://192.168.56.10/  -> aviso de certificado, y con "continuar" carga
```

**Nadie configuró el certificado.** Al instalar `mod_ssl`, un servicio de una sola ejecución (`httpd-init.service`) genera el par certificado + clave en `/etc/pki/tls/` y `ssl.conf` ya viene apuntando ahí. Por eso el HTTPS funciona con un `dnf install` y un `restart`.

**Por qué `curl` se queja.** `subject` e `issuer` son iguales: el certificado se firmó a sí mismo. No hay ninguna autoridad conocida que responda por él, así que `curl` corta la conexión. El `-k` dice "seguí igual" — sirve para probar, nunca para producción. Lo que resuelve esto de verdad es un certificado emitido por una CA (interna de la organización, o pública como Let's Encrypt).

**Ojo con el último `curl -k -H "Host: intranet..."`:** devuelve el **portal**, no la intranet. El virtual host de la intranet se definió solo para `*:80`; en el 443 solo existe el que trae `ssl.conf`. Para tener la intranet por HTTPS habría que agregarle su propio `<VirtualHost *:443>`.

**El 443 se abre en las dos zonas**, igual que el 80: la red host-only entra por `internal` y la NAT por `public`.
