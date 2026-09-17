# Reto — Solución

El ticket pide tres capas: puerto (SELinux), carpeta (SELinux) y firewall. Cualquier orden sirve; este es el natural.

```bash
# 1. Puerto 8082
echo "Listen 8082" | sudo tee /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
sudo ausearch -m AVC -ts recent | tail -1
sudo semanage port -a -t http_port_t -p tcp 8082
sudo semanage port -m -t http_port_t -p tcp 8082
sudo systemctl restart httpd
systemctl is-active httpd
```
El `restart` falla (`name_bind`, `src=8082`). El `-a` responde `ValueError: Port tcp/8082 already defined` (el 8082 es `us_cli_port_t` de fábrica): por eso el `-m`. Si en alguna VM el `-a` funciona sin error, el `-m` sobra: también vale.

```bash
# 2. Carpeta /sitio
sudo mkdir /sitio
echo "<h1>Portal institucional - rhel01 - puerto 8082</h1>" | sudo tee /sitio/index.html
sudo vim /etc/httpd/conf.d/web.conf
```
Cambiar las dos apariciones de `/web` por `/sitio`:
```
DocumentRoot "/sitio"
<Directory "/sitio">
    Require all granted
</Directory>
```
```bash
sudo apachectl configtest
sudo systemctl restart httpd
curl -I http://localhost:8082
sudo semanage fcontext -a -t httpd_sys_content_t "/sitio(/.*)?"
sudo restorecon -Rv /sitio
curl http://localhost:8082
```
El primer `curl -I` da `403` (`/sitio` es `default_t`); después de la regla y `restorecon`, el `<h1>`.

```bash
# 3. Firewall: 8082 en las dos zonas, 82 fuera de las dos
sudo firewall-cmd --permanent --add-port=8082/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=8082/tcp
sudo firewall-cmd --permanent --remove-port=82/tcp
sudo firewall-cmd --permanent --zone=internal --remove-port=82/tcp
sudo firewall-cmd --reload
```
Desde la computadora: `http://192.168.56.10:8082` carga.

## Verificación
```bash
getenforce
systemctl is-active httpd firewalld
sudo semanage port -l -C
sudo semanage fcontext -l -C
ls -Zd /sitio
curl http://localhost:8082
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
sudo semanage permissive -l
```
Esperado: `Enforcing`; `active` `active`; `http_port_t tcp 8082` y `82` en los puertos locales (el 82 puede quedar registrado: no molesta); `/sitio(/.*)?` y `/web(/.*)?` en los contextos locales; `unconfined_u:object_r:httpd_sys_content_t:s0 /sitio`; el `<h1>`; `8082/tcp` en las dos zonas y ningún `82/tcp`; nada bajo `Customized Permissive Types`.

## Errores que se ven

- `-a` en vez de `-m` para el 8082, y quedarse trabado en `already defined`.
- `chcon` y dejarlo así: `ls -Zd /sitio` da bien pero `semanage fcontext -l -C` no tiene `/sitio`. No cumple el requisito 2.
- Olvidar el bloque `<Directory "/sitio">`: 403 **de Apache**, no de SELinux. Se distingue porque `ausearch` no muestra ningún AVC nuevo sobre `/sitio`.
- Abrir el 8082 solo en `public`: por la ruta NAT no hay port forwarding para ese puerto, y por host-only cae en `internal`: no carga desde ningún lado. `--get-active-zones` primero.
- Olvidar `--reload`: `--permanent --list-ports` lo tiene, `--list-ports` no.
- `setenforce 0` "para probar" y olvidarse: descalifica. `getenforce` es la primera línea de la verificación por eso.
- Dejar `Listen 82` en un archivo y `Listen 8082` en otro dentro de `conf.d/`: Apache escucha en los dos; no cumple "ya no en el 82".
