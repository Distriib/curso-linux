# Lab 3.4 — Leer una denegación de punta a punta

Vamos a seguir el flujo completo: síntoma → AVC → traducción → decisión → corrección → verificación. Y a ver la alternativa profesional a `setenforce 0`.

---

## Parte 1 — Provocar el problema

**Un archivo de configuración preparado en /root y movido al sitio. ¿Qué va a pasar?**

```bash
echo "parametros internos del portal" | sudo tee /root/config.txt
sudo mv /root/config.txt /web/
curl -I http://localhost:82/config.txt
```

**Comprobar:** `HTTP/1.1 403 Forbidden`.

---

## Parte 2 — El AVC crudo

**¿Qué negó SELinux, a quién, sobre qué?**

```bash
sudo ausearch -m AVC -ts recent | tail -1
```

**Comprobar:**
```
type=AVC msg=audit(...): avc:  denied  { getattr } for  pid=... comm="httpd" path="/web/config.txt" ... scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file permissive=0
```
Leer con la tabla: `comm=httpd` intentó `getattr` sobre un `file` cuyo `tcontext` es `admin_home_t`. Un archivo de `/root` dentro de `/web`. Decisión: etiqueta mal → `restorecon`.

---

## Parte 3 — La versión traducida

**¿Cómo lo dice en palabras, y qué recomienda?**

```bash
sudo journalctl -t setroubleshoot --since "5 min ago" --no-pager
```

Copiar el código largo del final de la línea (lo que sigue a `sealert -l`) y pedir el informe:
```bash
sudo sealert -l CODIGO
```

Cada uno con su propio código.

**Comprobar:**
```
... setroubleshoot[...]: SELinux is preventing /usr/sbin/httpd from getattr access on the file /web/config.txt. For complete SELinux messages run: sealert -l 4c1f7a2e-...

SELinux is preventing /usr/sbin/httpd from getattr access on the file /web/config.txt.

*****  Plugin restorecon (99.5 confidence) suggests   ************************

If you want to fix the label.
/web/config.txt default label should be httpd_sys_content_t.
Then you can run restorecon.
Do
# /sbin/restorecon -v /web/config.txt

*****  Plugin catchall (1.49 confidence) suggests   **************************
...
```
Las sugerencias vienen ordenadas por confianza. La de 99.5 es la correcta. La de `catchall` con `audit2allow` **siempre aparece** y **casi nunca** es la correcta: le daría a Apache permiso para leer archivos de `/root` para siempre.

---

## Parte 4 — Corregir y verificar

**¿Se arregló?**

```bash
sudo restorecon -v /web/config.txt
curl http://localhost:82/config.txt
```

**Comprobar:**
```
Relabeled /web/config.txt from unconfined_u:object_r:admin_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
parametros internos del portal
```

---

## Parte 5 — Permissive para un solo servicio

**Una aplicación nueva da problemas y hay que seguir operando. ¿Cómo lo hago sin apagar SELinux para todos?**

```bash
sudo semanage permissive -a httpd_t
sudo semanage permissive -l
getenforce
sudo semanage permissive -d httpd_t
```

(El primer comando tarda unos segundos.)

**Comprobar:**
```
Customized Permissive Types

httpd_t

Builtin Permissive Types
...
Enforcing
```
El sistema sigue en `Enforcing`; solo `httpd_t` dejaba de ser bloqueado (y seguía registrando). El resto de los servicios, protegidos. Se quita con `-d` cuando se resolvió.

---

# Solución — todos los comandos

```bash
# Parte 1 — provocar el problema
echo "parametros internos del portal" | sudo tee /root/config.txt
sudo mv /root/config.txt /web/
curl -I http://localhost:82/config.txt        # 403 Forbidden

# Parte 2 — el AVC crudo
sudo ausearch -m AVC -ts recent | tail -1

# Parte 3 — la versión traducida
sudo journalctl -t setroubleshoot --since "5 min ago" --no-pager
```
De esa salida, copiar el código largo que sigue a `sealert -l` (es distinto en cada VM) y pedir el informe:
```bash
sudo sealert -l EL-CODIGO-QUE-TE-SALIO
```
```bash
# Parte 4 — corregir y verificar
sudo restorecon -v /web/config.txt
curl http://localhost:82/config.txt

# Parte 5 — permissive para un solo servicio (el primero tarda unos segundos)
sudo semanage permissive -a httpd_t
sudo semanage permissive -l
getenforce                                    # sigue diciendo Enforcing
sudo semanage permissive -d httpd_t
```

**El método, siempre igual:** síntoma → `ausearch` → mirar `tcontext` → decidir con la tabla → corregir → verificar con el mismo comando que falló.

**De todo el AVC, el campo que resuelve el caso es `tcontext`:** dice qué etiqueta tenía la cosa que el servicio quiso tocar. Acá decía `admin_home_t` dentro de `/web` — un archivo venido de `/root`, etiqueta mal → `restorecon`.

**Sobre `sealert`:** las sugerencias vienen ordenadas por confianza. La de ~99 % suele ser la correcta. La de `catchall` con `audit2allow` **aparece siempre** y casi nunca es la buena: acá le habría dado a Apache permiso permanente para leer archivos de `/root`.

**`semanage permissive -a httpd_t`** es la alternativa profesional a `setenforce 0`: deja de bloquear **un solo** servicio mientras se investiga, sin desproteger el resto del sistema. Se quita con `-d` en cuanto se resolvió — y el reto de hoy verifica que no quede ninguno puesto.
