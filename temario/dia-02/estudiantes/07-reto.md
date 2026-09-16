# Reto — Ticket #2026-0142

**Solos, sin ayuda.** Si no se termina en clase, queda de tarea.

El área de seguridad manda un log de acceso de 200 líneas (mismo formato de hoy: `FECHA HORA NIVEL USUARIO SERVICIO MENSAJE`) y pide lo siguiente.

## Preparar

El instructor pega en el chat el bloque que genera `~/reto/acceso.log`. Copiarlo entero, pegarlo, y verificar:

```bash
wc -l ~/reto/acceso.log
md5sum ~/reto/acceso.log
```

## Lo que piden

1. ¿Cuántas líneas contienen `FAIL`?
2. ¿Cuál es la última línea `WARN` del archivo (en el orden del archivo)?
3. Crear `~/reto/ana.log` únicamente con las líneas del usuario `ana`, ordenadas por fecha y hora.
4. Empaquetar y comprimir la carpeta `~/reto` en `~/backups/reto-AAAA-MM-DD.tar.gz`, con la fecha de hoy generada con `date` (no escrita a mano). Comprobar listando su contenido.
5. Crear un enlace simbólico `~/ultimo-backup` que apunte a ese `.tar.gz`, de modo que `tar -tzf ~/ultimo-backup` funcione desde cualquier carpeta.

**Bonus:** ¿qué servicio acumula más `ERROR` + `FAIL`? ¿Y cuál es la línea `WARN` más reciente en el tiempo (no en el archivo)?

## Entrega

Pegar en el chat la salida completa de:

```bash
grep -cw FAIL ~/reto/acceso.log; grep -w WARN ~/reto/acceso.log | tail -1; wc -l ~/reto/ana.log; ls -l ~/ultimo-backup; tar -tzf ~/ultimo-backup
```
