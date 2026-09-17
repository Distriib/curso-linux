# Comandos — SELinux: qué es

## El problema que resuelve

Los permisos `rwx` son **DAC**: el dueño decide, y root puede todo. Si alguien explota Apache y consigue ejecutar código, ese código corre como el usuario `apache` y va a intentar leer `/etc/shadow`, escribir en `/home`, conectarse afuera. Con solo `rwx`, varias de esas cosas funcionan.

SELinux agrega **MAC**: una política, escrita por Red Hat, dice qué puede hacer **cada proceso** según su **tipo**, sin importar quién sea el usuario. Apache (`httpd_t`) solo puede leer contenido web (`httpd_sys_content_t`), escuchar en puertos web (`http_port_t`) y poco más. El proceso comprometido sigue comprometido, pero encerrado en una habitación chica.

| | DAC (`rwx`) | MAC (SELinux) |
|---|---|---|
| Quién decide | el dueño del archivo | la política |
| root | puede todo | también está sujeto |
| Se ve con | `ls -l` | `ls -Z` |

Las dos comprobaciones ocurren; **ambas** tienen que aprobar.

## Modos

| Modo | Aplica la política | Registra lo denegado | Cuándo |
|---|---|---|---|
| `Enforcing` | sí | sí | producción, siempre |
| `Permissive` | **no** | sí | diagnóstico: unos segundos, nunca días |
| `Disabled` | no hay política | no | nunca |

```bash
getenforce
sestatus
```

- `sudo setenforce 0` / `sudo setenforce 1`: cambia el modo **hasta el próximo reinicio**.
- `/etc/selinux/config`: el modo con el que arranca. No se toca hoy.
- "Si en permissive funciona, es SELinux." Después se vuelve a enforcing **antes** de corregir.

## Contextos: la etiqueta

Todo archivo, proceso y puerto tiene una etiqueta de cuatro campos:

```
system_u:object_r:httpd_sys_content_t:s0
   │        │             │           └ nivel (no importa hoy)
   │        │             └ TIPO: lo único que importa el 95 % del tiempo (termina en _t)
   │        └ rol (no importa hoy)
   └ usuario SELinux (no importa hoy)
```

| Comando | Muestra la etiqueta de |
|---|---|
| `ls -Z archivo` | archivos |
| `ls -Zd carpeta` | la carpeta misma, no su contenido |
| `ps -eZ` | procesos |
| `id -Z` | tu sesión |

```bash
id -Z
ps -eZ | grep httpd
ls -Z /var/www/html/
```

## Tipos que hay que reconocer a la vista

| Tipo | Qué es |
|---|---|
| `httpd_t` | el proceso Apache |
| `httpd_sys_content_t` | contenido web que Apache puede leer |
| `http_port_t` | puertos donde Apache puede escuchar (80, 443, 8008…) |
| `reserved_port_t` | puertos menores a 1024 sin dueño (el 82) |
| `admin_home_t` | `/root` y lo que se crea ahí |
| `user_home_dir_t` / `user_home_t` | `/home/usuario` y lo que hay adentro |
| `default_t` | carpeta nueva en `/` sin regla: **ningún** servicio la lee |
| `shadow_t` | `/etc/shadow` |
| `tmp_t` | `/tmp` |
| `unconfined_t` | tu shell: a vos SELinux no te limita; limita a los servicios |

## De dónde sale la etiqueta de un archivo

- Un archivo nuevo **hereda el tipo de la carpeta** donde se crea.
- `cp` crea un archivo nuevo en el destino → hereda la etiqueta correcta.
- `mv` mueve el **mismo** archivo → conserva la etiqueta del origen.
- `restorecon archivo` le pone la etiqueta que la política dice que **debería** tener esa ruta.

El clásico: "lo copié a `/var/www/html` y funciona; lo moví y da 403".
