# Día 6 — Discos, sistemas de archivos y LVM

**4 horas**

## Qué vamos a hacer

1. **Cómo se organiza un disco** (25 min) — Partición, sistema de archivos, punto de montaje
2. **Particionar y montar** (55 min) — `parted`, formatear, y montaje permanente con `/etc/fstab`
3. **Memoria de intercambio** (incluido arriba) — Crear swap y dejarlo persistente
4. **LVM** (60 min) — Volúmenes que se amplían en caliente, sin apagar el servidor
5. **Stratis** (25 min) — Almacenamiento moderno con snapshots
6. **VDO y automontaje** (20 min) — Deduplicación y montaje bajo demanda

## Al terminar

Agregar un disco nuevo al servidor, dejarlo montado de forma permanente y
ampliar su capacidad sin interrumpir a los usuarios.

## Antes de empezar

Snapshot `dia05-fin` y **dos discos de 5 GB agregados** a la máquina virtual.
