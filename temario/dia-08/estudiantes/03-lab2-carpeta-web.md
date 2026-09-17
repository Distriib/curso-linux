# Lab 3.2 — La web desde una carpeta propia: /web

Vamos a servir el portal desde `/web` en vez de `/var/www/html`, ver por qué `chcon` no alcanza, y dejar una regla que sobreviva a todo.

---

## Parte 1 — La carpeta nueva y su etiqueta

**¿Con qué etiqueta nace una carpeta creada en `/`?**

```bash
sudo mkdir /web
echo "<h1>Portal institucional - servido desde /web</h1>" | sudo tee /web/index.html
ls -Zd /web
ls -Z /web
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
unconfined_u:object_r:default_t:s0 /web
unconfined_u:object_r:default_t:s0 index.html
```
`default_t` = "no tengo regla para esto". Ningún servicio confinado puede leerlo.

---

## Parte 2 — Apuntar Apache a /web

**¿Qué necesita Apache para servir desde otra carpeta?**

```bash
sudo vim /etc/httpd/conf.d/web.conf
```
Escribir estas cuatro líneas y guardar (`Esc` `:wq`):
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

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Syntax OK
HTTP/1.1 403 Forbidden
...
```
La configuración es correcta, la carpeta es `755`, el archivo `644`… y 403.

---

## Parte 3 — Confirmar con el AVC

**¿Es SELinux?**

```bash
sudo ausearch -m AVC -ts recent | tail -1
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
type=AVC msg=audit(...): avc:  denied  { getattr } for  pid=... comm="httpd" path="/web/index.html" ... scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```
`tcontext=...default_t` en una carpeta propia → no hay regla. (Puede salir sobre `/web` con `tclass=dir`: mismo diagnóstico.)

---

## Parte 4 — La tentación: chcon

**¿Por qué `chcon` no es la solución?**

```bash
sudo chcon -R -t httpd_sys_content_t /web
curl -I http://localhost:82
sudo restorecon -Rv /web
curl -I http://localhost:82
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
HTTP/1.1 200 OK
...
Relabeled /web from unconfined_u:object_r:httpd_sys_content_t:s0 to unconfined_u:object_r:default_t:s0
Relabeled /web/index.html from unconfined_u:object_r:httpd_sys_content_t:s0 to unconfined_u:object_r:default_t:s0
HTTP/1.1 403 Forbidden
...
```
`chcon` arregló; `restorecon` lo deshizo, porque la tabla sigue diciendo que `/web` es `default_t`. Un reetiquetado nocturno, una actualización o un compañero aplicado rompen la web.

---

## Parte 5 — La solución: regla + aplicar

**¿Cómo lo dejo para siempre?**

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
sudo semanage fcontext -l -C
sudo restorecon -Rv /web
ls -Z /web
curl http://localhost:82
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
SELinux fcontext                                   type               Context

/web(/.*)?                                         all files          system_u:object_r:httpd_sys_content_t:s0
Relabeled /web from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /web/index.html from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
<h1>Portal institucional - servido desde /web</h1>
```
Ahora `restorecon` **confirma** la etiqueta en vez de deshacerla.
