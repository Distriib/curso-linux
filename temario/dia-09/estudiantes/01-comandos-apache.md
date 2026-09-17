# Comandos — Apache: varios sitios y HTTPS

## Cómo está armado

```bash
ls /etc/httpd/
ls /etc/httpd/conf.d/
grep -n IncludeOptional /etc/httpd/conf/httpd.conf
```

| Ruta | Qué es |
|---|---|
| `/etc/httpd/conf/httpd.conf` | la configuración general: `Listen 80`, `DocumentRoot "/var/www/html"` |
| `/etc/httpd/conf.d/*.conf` | un archivo por sitio; se cargan **en orden alfabético** |
| `/var/www/html` | la carpeta del sitio por defecto |
| `/var/log/httpd/access_log` y `error_log` | quién entró, qué falló |

Antes de reiniciar, **siempre**:
```bash
sudo apachectl configtest
```
Un error de sintaxis deja el servicio caído. `Syntax OK` = se puede recargar.

## Un servidor, varios sitios: virtual host

Una sola IP y un solo puerto sirven varios sitios. Apache mira el **nombre** que pidió el navegador (la cabecera `Host:`) y elige el sitio cuyo `ServerName` coincide.

```
<VirtualHost *:80>                      ← escucha en cualquier IP, puerto 80
    ServerName intranet.lab.local       ← responde a este nombre
    ServerAlias intranet                ← y también a este
    DocumentRoot /var/www/intranet      ← sirve esta carpeta
    ErrorLog logs/intranet-error_log    ← logs/ = /var/log/httpd/
    CustomLog logs/intranet-access_log combined
    <Directory /var/www/intranet>
        Options Indexes                 ← sin index.html, muestra la lista de archivos
        Require all granted             ← cualquiera puede entrar
    </Directory>
</VirtualHost>
```

Si ningún nombre coincide, Apache sirve **el primer virtual host cargado** (por orden alfabético). Por eso el sitio por defecto va en `00-default.conf`.

| Comando | Qué hace |
|---|---|
| `curl http://localhost/` | pide el sitio sin nombre → el primero |
| `curl -H "Host: intranet.lab.local" http://localhost/` | pide el sitio **intranet**, sin tocar DNS |
| `sudo apachectl -S` | lista los virtual hosts cargados y cuál es el `default server` |
| `sudo systemctl reload httpd` | relee la configuración sin cortar conexiones |

SELinux: todo lo que se crea dentro de `/var/www` nace con `httpd_sys_content_t`, que es lo que Apache puede leer.

## HTTPS

| Pieza | Qué es |
|---|---|
| `sudo dnf install mod_ssl` | el módulo de HTTPS; agrega `conf.d/ssl.conf` con `Listen 443` |
| `/etc/pki/tls/certs/localhost.crt` | el certificado (público) |
| `/etc/pki/tls/private/localhost.key` | la clave privada: `600`, de root |
| `httpd-init` | servicio que genera ese certificado la primera vez que arranca `httpd` |
| `sudo openssl x509 -in cert -noout -subject -issuer -dates` | a nombre de quién, quién lo firmó, cuándo vence |
| `curl https://localhost/` | falla: nadie externo respalda el certificado |
| `curl -k https://localhost/` | `-k` = no validar el certificado (solo para pruebas) |

Un certificado **autofirmado** cifra igual que uno real, pero el navegador avisa que no confía en él. En producción lo firma una autoridad certificadora que el navegador ya conoce.

## Firewall: hoy todo se abre dos veces

```bash
sudo firewall-cmd --get-active-zones
```

| Zona | Por dónde entra |
|---|---|
| `public` | el port forwarding (`localhost:2222`, `localhost:8080`) |
| `internal` | la red host-only (`192.168.56.0/24`) |

Lo que se publica hoy se agrega en **las dos**: `--add-service=X` y `--zone=internal --add-service=X`, siempre con `--permanent` y al final `--reload`.
