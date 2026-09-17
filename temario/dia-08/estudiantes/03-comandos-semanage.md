# Comandos — SELinux: corregirlo con `semanage`

Hasta acá corregimos etiquetas que la política ya conocía (`restorecon`). Ahora agregamos **reglas locales** para casos legítimos: un puerto no estándar, una carpeta propia, una función apagada. La herramienta es `semanage`. Lo que hace es **persistente**: sobrevive a reinicios y reetiquetados.

## Puertos — `semanage port`

Cada puerto tiene un tipo. Apache solo puede escuchar en `http_port_t`.

```bash
sudo semanage port -l | grep -w http_port_t
```

| Comando | Qué hace |
|---|---|
| `sudo semanage port -a -t http_port_t -p tcp 82` | **agregar**: el 82 pasa a ser `http_port_t` |
| `sudo semanage port -m -t http_port_t -p tcp 8080` | **modificar**: cuando el puerto ya tiene otro tipo (`already defined`) |
| `sudo semanage port -d -t http_port_t -p tcp 82` | borrar lo agregado |
| `sudo semanage port -l -C` | solo lo que agregaste vos |

## Carpetas — `semanage fcontext` + `restorecon`

La política es una tabla **ruta → tipo** (`/var/www(/.*)? → httpd_sys_content_t`). `restorecon` consulta esa tabla. Una carpeta nueva en `/` no está en la tabla: queda `default_t` y Apache no la lee.

```
sudo semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
                        │  │                       └ la carpeta y todo lo de adentro (se escribe siempre así, entre comillas)
                        │  └ el tipo
                        └ agregar la regla a la tabla

sudo restorecon -Rv /web
                 └ aplicar la tabla a la carpeta y a todo su contenido (R), mostrando lo que cambia (v)
```

**Dos pasos, siempre:** la regla y aplicarla.

| Comando | Qué hace |
|---|---|
| `sudo semanage fcontext -l -C` | solo tus reglas |
| `sudo semanage fcontext -d "/web(/.*)?"` | borrar la regla |
| `sudo chcon -R -t httpd_sys_content_t /web` | cambia la etiqueta **sin tocar la tabla**: se pierde con el próximo `restorecon`. Solo para probar |

## Booleanos — interruptores

Partes de la política que vienen apagadas y se prenden sin escribir reglas.

| Comando | Qué hace |
|---|---|
| `getsebool -a` | todos los interruptores |
| `getsebool httpd_can_network_connect` | uno |
| `sudo setsebool -P httpd_can_network_connect on` | prenderlo **persistente** (sin `-P` se pierde al reiniciar) |
| `sudo semanage boolean -l -C` | los que cambiaste vos |

`httpd_can_network_connect`: Apache puede conectarse a otro servidor (proxy inverso). `httpd_enable_homedirs`: publicar la carpeta `public_html` del home de cada usuario.

## Leer una denegación (AVC)

Cada denegación queda en `/var/log/audit/audit.log`:

```
avc:  denied  { getattr }  for  pid=2411 comm="httpd" path="/web/config.txt"
      scontext=system_u:system_r:httpd_t:s0  tcontext=unconfined_u:object_r:admin_home_t:s0
      tclass=file permissive=0
```

| Campo | Pregunta que responde |
|---|---|
| `{ getattr }` / `{ read }` / `{ name_bind }` | ¿qué operación se negó? |
| `comm=` | ¿qué programa? |
| `scontext=` | ¿qué tipo de proceso lo intentó? → `httpd_t` |
| `tcontext=` | ¿qué etiqueta tenía la cosa? → **acá está casi siempre la pista** |
| `tclass=` | ¿qué era? `file`, `dir`, `tcp_socket` |
| `path=` / `src=` | ¿qué archivo? ¿qué puerto? |

| Herramienta | Qué da |
|---|---|
| `sudo ausearch -m AVC -ts recent` | las denegaciones de los últimos 10 minutos, crudas |
| `sudo journalctl -t setroubleshoot --since "5 min ago" --no-pager` | la misma denegación en una frase, con un código |
| `sudo sealert -l CODIGO` | el informe: qué pasó y qué hacer, ordenado por confianza |

## Tabla de decisión

| Lo que dice el AVC | Qué pasó | Qué hacer |
|---|---|---|
| `tclass=file`, `tcontext` con `admin_home_t` o `user_home_t`, en una carpeta estándar (`/var/www/html`) | archivo movido | `sudo restorecon -Rv /ruta` |
| `tclass=file` o `dir`, `tcontext=default_t`, carpeta propia (`/web`) | no hay regla | `semanage fcontext -a ...` + `restorecon -Rv` |
| `tclass=tcp_socket`, `{ name_bind }`, `src=PUERTO` | puerto sin etiqueta | `semanage port -a -t http_port_t -p tcp PUERTO` (`-m` si ya está definido) |
| `tclass=tcp_socket`, `{ name_connect }` | el servicio quiere **salir** a otro servidor | un booleano (`httpd_can_network_connect`) |
| `sealert` sugiere un booleano con confianza alta | función apagada | `setsebool -P nombre on` |

Lo que **no** está en la tabla: `setenforce 0` como solución, `SELINUX=disabled`, `chcon` en producción.

## Permissive para un solo servicio

Cuando una aplicación nueva da problemas y hay que seguir operando mientras se analiza:

| Comando | Qué hace |
|---|---|
| `sudo semanage permissive -a httpd_t` | solo Apache deja de ser bloqueado (sigue registrando); el resto sigue protegido |
| `sudo semanage permissive -l` | ver cuáles están así |
| `sudo semanage permissive -d httpd_t` | volver a la normalidad |
