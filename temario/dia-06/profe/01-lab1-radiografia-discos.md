# Lab — Radiografía de los discos (comandos)

Modo guiado: vos tipeás, ellos tipean, todos ven la salida, hacés la pregunta, esperás, y después das la respuesta. Nombrá a alguien para que responda. No se modifica nada. Vos en UTM: `vda`, `vdb`, `vdc`.

## Parte 1 — ¿Cuántos discos hay y cuáles están vacíos?
```bash
lsblk
```
Respuesta: tres discos (`sda`, `sdb`, `sdc`; `sr0` es el CD). `sdb` y `sdc` están vacíos: `disk` de 5G sin particiones debajo y sin `MOUNTPOINTS`. `sda` es el del sistema.

Si a alguien le falta un disco, lo agrega ahora con la VM apagada (los pasos están en la pestaña de estudiantes); los demás siguen, y cuando vuelva repite `lsblk`. Lo importante es que tenga dos discos de 5G vacíos, se llamen como se llamen.

## Parte 2 — ¿Dónde está `/`? ¿Y qué hay en `sda2`?
```bash
lsblk -f
```
Respuesta: `/` está montado en `rhel-root`, un volumen LVM, no en una partición. `sda2` dice `LVM2_member`: no tiene sistema de archivos, es un volumen físico de LVM. `sdb` y `sdc` no tienen `FSTYPE` ni `UUID`: sin firma.

## Parte 3 — ¿Cuánto espacio libre tiene el servidor?
```bash
df -h
```
Respuesta: lo que diga la línea de `/` (unos 17G en total). `sdb` y `sdc` no aparecen porque `df` solo muestra lo montado. Las líneas `tmpfs` y `devtmpfs` son memoria, no disco: ignorarlas.

## Parte 4 — ¿Tienen algo escrito los discos nuevos?
```bash
sudo blkid /dev/sdb /dev/sdc
sudo fdisk -l /dev/sdb
```
Respuesta: `blkid` no imprime nada (sin firma). En `fdisk -l` no hay línea `Disklabel type`: sin tabla de particiones. Para comparar, `sudo fdisk -l /dev/sda` sí la tiene (`dos` o `gpt`).

## Parte 5 — ¿Qué lee el sistema al arrancar para montar todo esto?
```bash
cat /etc/fstab
```
Respuesta: tres líneas (cuatro en UTM, con `/boot/efi`): qué dispositivo, en qué carpeta, de qué tipo, con qué opciones. `/boot` va por UUID. Es el archivo del Lab 2.3; hoy le agregamos cinco líneas. Señalar el comentario del propio archivo que pide `systemctl daemon-reload` después de editarlo.
