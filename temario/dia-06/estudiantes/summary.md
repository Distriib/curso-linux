# Día 6 — Discos, sistemas de archivos y LVM

## Qué vamos a hacer

1. **Cómo se organiza un disco** — Disco, partición, sistema de archivos y punto de montaje; qué hizo el instalador con el nuestro
2. **Particionar, formatear y montar** — `parted`, `fdisk`, `mkfs`, montaje permanente con `/etc/fstab`, y swap
3. **LVM** — Volúmenes que se amplían en caliente, sin apagar el servidor
4. **Stratis** — Un pool con snapshots que se amplía con un solo comando
5. **Automontaje y VDO** — Montar bajo demanda y deduplicar: qué son y cuándo se usan

Hasta ayer el servidor era el que salió de la instalación: un disco, y listo. Hoy le agregamos discos, los repartimos y los hacemos crecer sin apagarlo.

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 15 | **Comandos:** disco → partición → sistema de archivos → punto de montaje; GPT, XFS, UUID, qué es LVM (explicado en consola) |
| 1 | 10 | **Lab 1.1:** radiografía de los discos — guiado: todos tipean, después leemos la salida; acá se confirma que están los dos discos nuevos |
| 2 | 10 | **Comandos:** `parted`, `fdisk`, `mkfs`, `mount`, `/etc/fstab` y swap (explicado en consola) |
| 2 | 20 | **Lab 2.1:** particionar `sdb` y `sdc` — yo hago las dos primeras particiones, ustedes las otras dos, foto |
| 2 | 15 | **Lab 2.2:** formatear, etiquetar y montar a mano — guiado: todos tipean, foto de cada parte |
| 2 | 15 | **Lab 2.3:** montaje permanente con `/etc/fstab` — yo hago el primero, ustedes lo repiten con su UUID, foto |
| 2 | 10 | **Lab 2.4:** swap en partición — yo hago el primero, ustedes lo repiten con su UUID, foto |
| — | 15 | Descanso |
| 3 | 10 | **Comandos:** LVM — PV, VG, LV y cómo se amplían (explicado en consola) |
| 3 | 20 | **Lab 3.1:** crear un volumen LVM y montarlo — yo hago el primer contrato, ustedes los demás, foto |
| 3 | 15 | **Lab 3.2:** ampliar en caliente — guiado: todos tipean, foto de cada parte |
| 3 | 10 | **Lab 3.3:** snapshot, renombrar y borrar — yo hago el primero, ustedes el resto, foto |
| 4 | 5 | **Comandos:** Stratis (explicado en consola) |
| 4 | 25 | **Lab 4.1:** pool Stratis con snapshot y ampliación — yo hago el primero, ustedes lo repiten con su UUID, foto |
| 5 | 10 | **Comandos:** automontaje y VDO — qué son y cuándo se usan (explicado en consola) |
| — | 0 | Reto individual — queda de tarea si no alcanza el reloj |
| — | 10 | Cierre y snapshot |
| — | 25 | Colchón (margen para imprevistos) |

---

## Al terminar

Agregar un disco nuevo al servidor, dejarlo montado de forma permanente y ampliar su capacidad sin interrumpir a los usuarios.

---

## Tarea

1. **Snapshot `dia06-fin`** con la VM apagada (`sudo poweroff`), con `/datos` montado y `vg_datos` intacto: el Día 7 escribe en `/backups` y el Día 9 comparte `/datos` por red. **No borrar** nada de lo que se armó hoy.
2. **Verificar los repositorios**, porque el Día 7 se instala software: `sudo dnf repolist` tiene que listar BaseOS y AppStream. Si dice `This system is not registered`: `sudo subscription-manager register`.
3. **El reto** (Ticket #2026-0614) si no se hizo en clase. Mandar la salida de la verificación por el chat.
