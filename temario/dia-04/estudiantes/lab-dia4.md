# Laboratorio — Día 4

Cuatro entregas. Cada una tiene un **objetivo** y una **foto** que hay que mandar al chat. Trabajan solos; los comandos que hacen falta están en las hojas de comandos de hoy.

---

# Entrega 1 — El proceso que come CPU

**La situación:** el servidor está lento. Algo llamado `actualizador-pgn` está consumiendo el procesador.

Para reproducirlo en su VM, corran estas dos líneas:

```bash
cp /usr/bin/yes ~/actualizador-pgn
~/actualizador-pgn > /dev/null &
```

**Objetivo:** dejar el servidor tranquilo otra vez.

1. Averiguar qué proceso está consumiendo la CPU, y cuánto.
2. Averiguar su número de proceso (PID) y qué usuario lo lanzó.
3. Bajarle la prioridad al mínimo, y comprobar en el monitor que ya no molesta.
4. Terminarlo.
5. Comprobar que ya no existe.
6. Borrar el archivo `~/actualizador-pgn`.

**Lo que mandás:** dos fotos.
- Una del monitor de procesos con `actualizador-pgn` consumiendo CPU.
- Otra, después de terminarlo, de la búsqueda del proceso sin resultados.

---

# Entrega 2 — Un servicio propio que resucita

**Objetivo:** que exista en el servidor un servicio llamado `reporte` que escriba la fecha en `/var/log/reporte.log` cada 30 segundos, que esté **activo**, que arranque **solo** al encender el servidor, y que **vuelva solo** si alguien lo mata.

**1.** Crear el archivo `/usr/local/bin/reporte.sh` con este contenido:

```bash
#!/bin/bash
while true; do
    date >> /var/log/reporte.log
    sleep 30
done
```

**2.** Darle permiso de ejecución. (Es el `chmod` de ayer.)

**3.** Crear el archivo `/etc/systemd/system/reporte.service` con este contenido:

```
[Unit]
Description=Reporte de la PGN

[Service]
ExecStart=/usr/local/bin/reporte.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

**4.** Avisarle a systemd que hay un archivo nuevo. **Sin este paso no pasa nada.**

**5.** Arrancar el servicio y dejarlo habilitado, con un solo comando.

**6.** Ver su estado y **anotar el número de PID** que aparece.

**7.** Matar ese PID a mano. Esperar cinco segundos. Volver a ver el estado.

**Lo que mandás:** dos fotos del estado del servicio — una antes del `kill` y otra después. Tienen que decir `active (running)` y `enabled` las dos, **con PID distinto**.

---

# Entrega 3 — Que todo sobreviva un reinicio

**Objetivo:** dejar el servidor de forma que, después de apagarlo y encenderlo, tres cosas sigan funcionando solas.

**1.** Instalar el servidor web (`httpd`) y dejarlo arrancando solo al encender el servidor.

**2.** Comprobar que responde: `curl http://localhost` tiene que devolver la página de prueba de Apache.

**3.** Hacer que los logs sobrevivan al reinicio. Hoy viven en la memoria y se borran. Se arregla creando la carpeta `/var/log/journal` y reiniciando el servicio `systemd-journald`.

**4.** Anotar cuántos arranques muestra `journalctl --list-boots`.

**5.** Reiniciar la VM entera.

**6.** Volver a entrar por SSH y comprobar, **sin arrancar nada a mano**:
   - que Apache responde
   - que el servicio `reporte` está corriendo
   - cuántos arranques muestra ahora `journalctl --list-boots`

**Lo que mandás:** una foto después del reinicio con las tres comprobaciones: el `curl` respondiendo, el estado de `reporte`, y la lista de arranques (tiene que mostrar **dos**).

---

# Entrega 4 — Instalar lo que Red Hat no trae

**La situación:** necesitan `htop`, un monitor de procesos más cómodo que `top`. No está en los repositorios de Red Hat.

**1.** Intentar instalarlo con `dnf`. Va a fallar. Leer el error.

**2.** Agregar el repositorio EPEL:

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

**3.** Comprobar que ahora aparece en la lista de repositorios.

**4.** Instalar `htop`. Abrirlo, mirarlo, y salir con `q`.

**5.** Averiguar de qué paquete salió el archivo `/usr/bin/htop`, y cuántos archivos instaló ese paquete en total.

**6.** Ver el historial de instalaciones del sistema.

**7.** Deshacer la última instalación con `dnf history undo`.

**8.** Comprobar que `htop` ya no existe.

**Lo que mandás:** una foto del historial de `dnf` mostrando la instalación y el `undo` debajo, y otra del intento de correr `htop` diciendo que no existe.
