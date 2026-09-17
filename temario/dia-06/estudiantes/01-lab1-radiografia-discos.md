# Lab 1.1 — Radiografía de los discos

Todos corren cada comando en su VM; después de cada parte, la leemos juntos y foto. Hoy en este lab **no se modifica nada**. En UTM: `vda`, `vdb`, `vdc` en vez de `sda`, `sdb`, `sdc`.

---

## Parte 1 — ¿Cuántos discos hay y cuáles están vacíos?

```bash
lsblk
```

**Comprobar:**
```
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk
├─sda1          8:1    0    1G  0 part /boot
└─sda2          8:2    0   19G  0 part
  ├─rhel-root 253:0    0   17G  0 lvm  /
  └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
sdb             8:16   0    5G  0 disk
sdc             8:32   0    5G  0 disk
sr0            11:0    1 1024M  0 rom
```
Tres discos. `sdb` y `sdc` son `disk` de 5G **sin nada debajo**: ni particiones ni punto de montaje.

Si te falta `sdb` o `sdc`: `sudo poweroff`, agregar el disco con la VM apagada (VirtualBox: Configuración → Almacenamiento → Controlador SATA → Agregar disco duro → Crear → VDI → 5 GB. UTM: Editar → Unidades → Nuevo → VirtIO → 5 GB), arrancar, reconectar por SSH y repetir `lsblk`.

---

## Parte 2 — ¿Dónde está `/`? ¿Y qué hay en `sda2`?

```bash
lsblk -f
```

**Comprobar:**
```
NAME          FSTYPE      FSVER    LABEL UUID   FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1        xfs                        ...       ...    ...  /boot
└─sda2        LVM2_member LVM2 001       ...
  ├─rhel-root xfs                        ...       ...    ...  /
  └─rhel-swap swap        1              ...                   [SWAP]
sdb
sdc
```
`/` no está en una partición: está en el volumen LVM `rhel-root`. `sda2` no tiene sistema de archivos, dice `LVM2_member`: está entregada a LVM. `sdb` y `sdc` no tienen ni `FSTYPE` ni `UUID`.

---

## Parte 3 — ¿Cuánto espacio libre tiene el servidor?

```bash
df -h
```

**Comprobar:**
```
/dev/mapper/rhel-root    17G  ...   ...   ...% /
/dev/sda1               960M  ...   ...   ...% /boot
```
`sdb` y `sdc` **no aparecen**: `df` solo muestra lo que está montado.

---

## Parte 4 — ¿Tienen algo escrito los discos nuevos?

```bash
sudo blkid /dev/sdb /dev/sdc
sudo fdisk -l /dev/sdb
```

**Comprobar:**
```
Disk /dev/sdb: 5 GiB, 5368709120 bytes, 10485760 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
`blkid` no imprime nada: no hay firma. `fdisk` no muestra la línea `Disklabel type`: no hay tabla de particiones. En UTM el modelo dice `Virtio Block Device`.

---

## Parte 5 — ¿Qué lee el sistema al arrancar para montar todo esto?

```bash
cat /etc/fstab
```

**Comprobar:**
```
/dev/mapper/rhel-root   /                       xfs     defaults        0 0
UUID=5d2f8b1a-...       /boot                   xfs     defaults        0 0
/dev/mapper/rhel-swap   none                    swap    defaults        0 0
```
Tres líneas (cuatro en UTM, con `/boot/efi`): qué, dónde, de qué tipo. `/boot` va por UUID. Hoy le agregamos cinco líneas a este archivo.
