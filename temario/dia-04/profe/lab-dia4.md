# Laboratorio Día 4 — respuestas

Cuatro entregas. Ellos trabajan solos; vos mirás el chat. Si alguien se traba, una pista, no el comando. Cuando se cumple el tiempo, mostrás esto y todos quedan parejos.

Tiempos: **25 · 30 · 20 · 25** minutos.

---

# Entrega 1 — El proceso que come CPU (25 min)

```bash
cp /usr/bin/yes ~/actualizador-pgn
~/actualizador-pgn > /dev/null &
```

**Qué es `yes`:** un programa que imprime la letra `y` sin parar, para siempre. No sirve para nada salvo para quemar CPU, que es justo lo que queremos. Al copiarlo con otro nombre, en el monitor aparece como `actualizador-pgn` y parece un proceso legítimo.

**El `&` del final:** lo manda a segundo plano, así les devuelve el prompt y pueden seguir trabajando.

### Solución

```bash
top
```
Se ve `actualizador-pgn` con 100% de CPU. `q` para salir.

```bash
ps -ef | grep actualizador
```
Muestra el usuario (`student`), el PID, y la ruta completa.

```bash
renice -n 19 -p PID
top
```
`19` es la prioridad más baja. En `top` sigue apareciendo, pero ya no estorba: cualquier otra cosa le pasa por encima. Un usuario normal **solo puede bajar** la prioridad, nunca subirla — para subirla hace falta root.

```bash
kill PID
ps -ef | grep actualizador
rm ~/actualizador-pgn
```
`kill` a secas pide por favor (señal 15) y `yes` obedece. Si un proceso no muriera, `kill -9 PID` lo mata sin preguntar — pero es el último recurso: no le da tiempo a cerrar archivos.

### Qué decir al cerrar

*"El nombre de un proceso no significa nada. Cualquiera puede llamar a su programa `actualizador-pgn` o `systemd-helper`. Lo que importa es qué hace, quién lo lanzó y desde dónde salió. Así se esconde el malware en un servidor."*

### Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `renice: failed to set priority: Permission denied` | Intentaron bajar el número (subir prioridad). Solo root puede |
| `kill: (1234) - No such process` | Ya lo mataron, o se equivocaron de PID |
| `top` no responde | Están adentro; `q` para salir |
| El proceso vuelve a aparecer | Corrieron la línea de `cp` y lanzamiento dos veces |

---

# Entrega 2 — Un servicio propio que resucita (30 min)

### Solución

```bash
sudo vim /usr/local/bin/reporte.sh
```
`i`, pegar el script, `Esc`, `:wq`.

```bash
sudo chmod +x /usr/local/bin/reporte.sh
sudo vim /etc/systemd/system/reporte.service
```
`i`, pegar la unidad, `Esc`, `:wq`.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now reporte
systemctl status reporte
```

Sale `active (running)`, `enabled`, y una línea `Main PID: 1234 (reporte.sh)`.

```bash
sudo kill 1234
systemctl status reporte
```
A los pocos segundos vuelve a decir `active (running)` **con otro PID**. En el `status` se ve la línea del reinicio automático.

```bash
sudo tail /var/log/reporte.log
```
Una fecha cada 30 segundos.

### Qué explicar de cada pieza

- **`daemon-reload`** — systemd tiene la lista de servicios cargada en memoria. Si creás un archivo nuevo y no avisás, systemd no se entera: `Unit reporte.service not found`. Es el paso que todos olvidan, incluido en el examen.
- **`enable --now`** — dos cosas en un comando: `start` (arrancá ahora) y `enable` (arrancá solo al encender el servidor). Son independientes: un servicio puede estar corriendo y no habilitado, o habilitado y apagado.
- **`Restart=always`** — la línea que hace que resucite. systemd vigila el proceso; si muere por lo que sea, lo vuelve a levantar. Es lo que se usa en producción para que un servicio caído no espere hasta que alguien lo note.
- **`WantedBy=multi-user.target`** — a qué momento del arranque se engancha. `multi-user` es "el servidor ya está listo, con red". Sin esta sección, `enable` no tiene dónde engancharlo.
- **El `while true` del script** — el servicio tiene que quedarse vivo. Si el script terminara enseguida, con `Restart=always` systemd lo relanzaría sin parar y terminaría bloqueándolo (`start request repeated too quickly`).

### Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `Unit reporte.service not found` | Falta `daemon-reload` |
| `status` dice `failed` con `code=exited, status=203/EXEC` | Falta el `chmod +x`, o la ruta del `ExecStart` está mal escrita |
| `failed` con `status=127` | El archivo no empieza con `#!/bin/bash` |
| Arranca pero no escribe el log | Miraron `/var/log/reporte.log` antes de que pasaran 30 segundos |
| `enabled` no aparece | Hicieron `start` sin `enable`, o falta la sección `[Install]` |
| El PID no cambia después del `kill` | Mataron otro PID (el del `sleep`, no el del script). Que usen el `Main PID` del `status` |

---

# Entrega 3 — Que todo sobreviva un reinicio (20 min)

### Solución

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
curl http://localhost
```
Devuelve el HTML de la página de prueba de Red Hat.

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
journalctl --list-boots
```
Ahora muestra **un** arranque.

```bash
sudo systemctl reboot
```
Se corta el SSH. Esperar un minuto y volver a entrar:

```bash
ssh -p 2222 student@localhost
curl http://localhost
systemctl status reporte
journalctl --list-boots
```
Apache responde, `reporte` está `active`, y la lista muestra **dos** arranques.

### Qué explicar

- **`start` vs `enable`** — esto es lo único que importa del bloque, y el reinicio es la única forma de entenderlo. `start` es ahora; `enable` es para siempre. Un servicio arrancado a mano y no habilitado desaparece en el próximo reinicio, y nadie se entera hasta que el servidor se reinicia solo una madrugada.
- **El journal** — es el sistema de logs de systemd: una base de datos donde queda todo lo que pasa en el servidor. Por defecto en RHEL vive **en la memoria**: al reiniciar se borra. Creando `/var/log/journal` pasa a guardarse en disco. Por eso la lista de arranques pasa de uno a dos: ahora se acuerda del anterior.
- **Por qué importa** — cuando investigás por qué se cayó un servidor anoche, si el journal no era persistente no hay nada que leer. Es de lo primero que se configura en un servidor de verdad.

### Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `curl: (7) Failed to connect` | No arrancaron `httpd`, o hicieron `enable` sin `--now` |
| Después del reinicio Apache no está | Hicieron `start` sin `enable` — **es el error que el lab quiere provocar**. Que lo arreglen con `enable` |
| `--list-boots` sigue mostrando uno | No reiniciaron `systemd-journald`, o crearon la carpeta mal escrita |
| No vuelve el SSH | La VM tarda; esperar un minuto más antes de preocuparse |

---

# Entrega 4 — Instalar lo que Red Hat no trae (25 min)

### Solución

```bash
sudo dnf install -y htop
```
Falla: `No match for argument: htop` / `Unable to find a match`.

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
dnf repolist
```
Ahora la lista incluye `epel` y `epel-cisco-openh264`.

```bash
sudo dnf install -y htop
htop
```
`q` para salir.

```bash
rpm -qf /usr/bin/htop
rpm -ql htop | wc -l
```
El primero dice de qué paquete salió ese archivo; el segundo, cuántos archivos trae el paquete.

```bash
sudo dnf history
sudo dnf history undo last
htop
```
El historial muestra las transacciones numeradas. Después del `undo`, `htop` responde `command not found`.

### Qué explicar

- **Qué es un repositorio** — el lugar de donde `dnf` baja los paquetes. No tiene nada que ver con git. BaseOS y AppStream son los de Red Hat, y se habilitaron al registrar la VM. EPEL es otro, de la comunidad, con lo que Red Hat no empaqueta.
- **Por qué `htop` no está en Red Hat** — Red Hat solo empaqueta lo que va a soportar durante diez años. `htop` no entra en ese compromiso. Por eso existe EPEL.
- **La advertencia para la institución** — los paquetes de EPEL **no tienen el soporte que la PGN está pagando**. En una VM de práctica da igual; en un servidor de producción, es una decisión que se consulta. Buen momento para decir por qué un administrador no instala cualquier cosa.
- **`rpm` vs `dnf`** — `dnf` va a internet, resuelve dependencias e instala. `rpm` solo consulta o maneja lo que ya está en la máquina. Por eso las preguntas del tipo "¿de qué paquete salió este archivo?" se contestan con `rpm`.
- **`dnf history undo`** — deshace una transacción completa, con todo lo que arrastró. Es la red de seguridad: si una actualización rompió algo, se revierte con un comando en vez de reinstalar a mano.

### Errores que vas a ver

| Pasa | Por qué |
|---|---|
| El `dnf install` de EPEL falla con error de red | La VM no tiene salida a internet. Comprobar con `curl -I https://dl.fedoraproject.org` |
| `dnf repolist` no muestra `epel` | El paquete `epel-release` no se instaló; repetir |
| `dnf history undo last` da error | Hay una transacción posterior; usar el número exacto de la lista: `sudo dnf history undo 12` |
| `htop` sigue funcionando después del `undo` | La sesión tiene la ruta en caché; `hash -r` o reconectar |
