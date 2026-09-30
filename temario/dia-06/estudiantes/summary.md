# Día 6 — Discos, sistemas de archivos y LVM

## Qué vamos a hacer

1. **Cómo se organiza un disco** — Disco, partición, sistema de archivos y punto de montaje; qué hizo el instalador con el nuestro
2. **Particionar, formatear y montar** — `parted`, `fdisk`, `mkfs`, montaje permanente con `/etc/fstab`, y swap
3. **LVM** — Volúmenes que se amplían en caliente, sin apagar el servidor
4. **Stratis** — Un pool con snapshots que se amplía con un solo comando
5. **Automontaje y VDO** — Montar bajo demanda y deduplicar: qué son y cuándo se usan

Hasta ayer el servidor era el que salió de la instalación: un disco, y listo. Hoy le agregamos discos, los repartimos y los hacemos crecer sin apagarlo.

## Los laboratorios del día

| Lab | Qué se hace |
|---|---|
| 1.1 | Radiografía de los discos: cuántos hay, qué tiene cada uno, qué está montado |
| 2.1 | Particionar `sdb` con `parted` y `sdc` con `fdisk` |
| 2.2 | Formatear, etiquetar y montar a mano |
| 2.3 | Montaje permanente con `/etc/fstab` |
| 2.4 | Swap en partición |
| 3.1 | Crear un volumen LVM y montarlo |
| 3.2 | Ampliar en caliente, sin desmontar nada |
| 3.3 | Snapshot, renombrar y borrar volúmenes |
| 4.1 | Un pool Stratis con snapshot y ampliación |

Cada lab termina con una sección **Solución** con todos los comandos. Mirarla después de intentarlo.

**Reto del día:** Ticket #2026-0614 — volumen de respaldos permanente, ampliado en caliente, más un archivo de swap.

---

## Al terminar

Agregar un disco nuevo al servidor, dejarlo montado de forma permanente y ampliar su capacidad sin interrumpir a los usuarios.

---

## Tarea

1. **Snapshot `dia06-fin`** con la VM apagada (`sudo poweroff`), con `/datos` montado y `vg_datos` intacto: el Día 7 escribe en `/backups` y el Día 9 comparte `/datos` por red. **No borrar** nada de lo que se armó hoy.
2. **Verificar los repositorios**, porque el Día 7 se instala software: `sudo dnf repolist` tiene que listar BaseOS y AppStream. Si dice `This system is not registered`: `sudo subscription-manager register`.
3. **El reto** (Ticket #2026-0614) si no se hizo en clase. Mandar la salida de la verificación por el chat.
