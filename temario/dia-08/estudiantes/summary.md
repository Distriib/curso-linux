# Día 8 — Seguridad: firewall, SELinux y hardening

## Qué vamos a hacer

1. **firewalld** — Zonas, servicios y puertos: abrir solo lo necesario, y solo para la red que corresponde
2. **SELinux: qué es** — Por qué existe, en qué modo está, y el error clásico del archivo movido
3. **SELinux: corregirlo** — Puertos, carpetas propias y booleanos con `semanage`; leer una denegación
4. **Hardening** — SSH solo con llaves, contraseñas fuertes con bloqueo, y cerrar lo que nadie pidió

Hasta ayer la pregunta era "cómo hago que el servidor haga X". Hoy es "cómo hago que **no** pueda hacer nada más que X, aunque lo ataquen".

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 15 | **Comandos:** firewalld — zonas, servicios, runtime vs permanent (explicado en consola) |
| 1 | 25 | **Lab 1.1:** publicar Apache a través del firewall — todos tipean, después leemos la salida; foto |
| 1 | 20 | **Lab 1.2:** la red de administración en su zona, y un Apache que no arranca — todos tipean; foto |
| 2 | 15 | **Comandos:** SELinux — modos, contextos, tipos (explicado en consola) |
| 2 | 10 | **Lab 2.1:** modos y contextos — probar permissive y volver; foto |
| 2 | 10 | **Lab 2.2:** el clásico: mv, 403 y restorecon; foto |
| — | 15 | Descanso |
| 3 | 10 | **Comandos:** semanage — puertos, carpetas, booleanos; leer un AVC (explicado en consola) |
| 3 | 10 | **Lab 3.1:** Apache en el puerto 82; foto |
| 3 | 15 | **Lab 3.2:** la web desde una carpeta propia; foto |
| 3 | 5 | **Lab 3.3:** booleanos; foto |
| 3 | 15 | **Lab 3.4:** leer una denegación de punta a punta; foto |
| 4 | 5 | **Comandos:** hardening — cuatro frentes (explicado en consola) |
| 4 | 15 | **Lab 4.1:** SSH solo con llaves, sin root, probado antes de cerrar la puerta; foto |
| 4 | 12 | **Lab 4.2:** contraseñas fuertes y bloqueo por intentos; foto |
| 4 | 8 | **Lab 4.3:** superficie: qué escucha, qué se cierra, parches solos; foto |
| — | 0 | Reto individual — queda de tarea |
| — | 10 | Cierre y snapshot |
| — | 25 | Colchón (margen para imprevistos) |

---

## Al terminar

Publicar un servicio abriendo solo lo necesario, resolver un bloqueo de SELinux sin apagarlo, y dejar el acceso remoto y las contraseñas endurecidos.

---

## Tarea

1. **El reto** (Ticket PGN-2026-0812) si no se hizo en clase. Mandar la salida de la verificación por el chat.

2. **Devolver Apache al estándar** (mañana se usa en el puerto 80 con `/var/www/html`), después del reto:
   ```bash
   sudo rm /etc/httpd/conf.d/puerto.conf /etc/httpd/conf.d/web.conf
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
