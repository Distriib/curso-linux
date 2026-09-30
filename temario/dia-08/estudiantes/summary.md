# Día 8 — Seguridad: firewall, SELinux y hardening

## Qué vamos a hacer

1. **firewalld** — Zonas, servicios y puertos: abrir solo lo necesario, y solo para la red que corresponde
2. **SELinux: qué es** — Por qué existe, en qué modo está, y el error clásico del archivo movido
3. **SELinux: corregirlo** — Puertos, carpetas propias y booleanos con `semanage`; leer una denegación
4. **Hardening** — SSH solo con llaves, contraseñas fuertes con bloqueo, y cerrar lo que nadie pidió

Hasta ayer la pregunta era "cómo hago que el servidor haga X". Hoy es "cómo hago que **no** pueda hacer nada más que X, aunque lo ataquen".

## Los laboratorios del día

| Lab | Qué se hace |
|---|---|
| 1.1 | Publicar Apache a través del firewall: runtime vs permanent |
| 1.2 | La red de administración en la zona `internal`, y un Apache que no arranca |
| 2.1 | Modos de SELinux y cómo leer contextos |
| 2.2 | El clásico: `mv`, 403 y `restorecon` |
| 3.1 | Apache en el puerto 82 con `semanage port` |
| 3.2 | La web desde una carpeta propia `/web`, y por qué `chcon` no alcanza |
| 3.3 | Booleanos: prender y apagar partes de la política |
| 3.4 | Leer una denegación de punta a punta, con `sealert` |
| 4.1 | SSH solo con llaves, sin root, probado antes de cerrar la puerta |
| 4.2 | Contraseñas fuertes y bloqueo por intentos fallidos |
| 4.3 | Superficie: qué escucha, qué se cierra, parches solos |

Cada lab termina con una sección **Solución** con todos los comandos seguidos.

**Reto del día:** Ticket PGN-2026-0812 — el portal en el puerto 8082 y en `/sitio`, sin apagar nada.

## Las dos rutas para ver la web desde tu computadora

| Ruta | Dirección | Por dónde entra |
|---|---|---|
| NAT (port forwarding) | `http://localhost:8080` | zona `public`, por la interfaz |
| Host-only | `http://192.168.56.10` | zona `internal` desde el Lab 1.2, por el origen |

Si `localhost:8080` no carga nunca, es que a tu VM le falta la regla de reenvío de puertos. Con la VM **apagada**, en VirtualBox: Configuración → Red → Adaptador 1 (NAT) → Avanzado → Reenvío de puertos → **+** → Host `8080`, Invitado `80`, protocolo TCP. En UTM: Configuración → Red → Reenvío de puertos → **+**, lo mismo. La ruta host-only (`192.168.56.10`) funciona sin esa regla y sirve igual para todos los labs.

---

## Al terminar

Publicar un servicio abriendo solo lo necesario, resolver un bloqueo de SELinux sin apagarlo, y dejar el acceso remoto y las contraseñas endurecidos.

---

## Tarea

1. **El reto** (Ticket PGN-2026-0812), que junta todo lo del día en un solo ejercicio.

2. **Devolver Apache al estándar** (mañana se usa en el puerto 80 con `/var/www/html`), después del reto.

   Primero mirar qué archivos hay:
   ```bash
   ls /etc/httpd/conf.d/
   ```
   Hay que borrar **todos los que creaste hoy**: `puerto.conf`, `web.conf`, y el que hayas usado en el reto para apuntar a `/sitio` (si le pusiste otro nombre, ese también). Los que vinieron de fábrica — `autoindex.conf`, `userdir.conf`, `welcome.conf`, `README` — **no se tocan**.
   ```bash
   sudo rm -f /etc/httpd/conf.d/puerto.conf /etc/httpd/conf.d/web.conf
   sudo systemctl restart httpd
   curl http://localhost
   sudo firewall-cmd --permanent --remove-port=82/tcp
   sudo firewall-cmd --permanent --zone=internal --remove-port=82/tcp
   sudo firewall-cmd --permanent --remove-port=8082/tcp
   sudo firewall-cmd --permanent --zone=internal --remove-port=8082/tcp
   sudo firewall-cmd --reload
   sudo firewall-cmd --get-active-zones
   ```
   Tiene que mostrar `<h1>Portal institucional - version 2</h1>`, y al final `internal` con `sources: 192.168.56.0/24` y `public` con las interfaces. Un `Warning: NOT_ENABLED` en algún `--remove-port` no es error: ese puerto ya estaba cerrado.

3. **Snapshot `dia08-fin`** con la VM apagada. Antes, verificar: `getenforce` dice `Enforcing` y `sudo sshd -T | grep -i permitrootlogin` dice `no`. **No borrar** la zona `internal` ni el hardening de SSH: se usan los Días 9 y 10.
