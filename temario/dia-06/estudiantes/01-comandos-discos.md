# 1 — Cómo se organiza un disco

Los comandos dicen `sdb` y `sdc` (VirtualBox). En UTM (Mac) los discos se llaman `vdb` y `vdc`, y las particiones `vdb1`, `vdb2`... Todo lo demás es igual.

## La cadena: cuatro pasos, cuatro comandos

| Paso | Qué es | Ejemplo | Comando |
|---|---|---|---|
| Disco | el dispositivo: una tira de bytes sin estructura | `/dev/sdb` | `lsblk` |
| Partición | un pedazo del disco, con inicio y fin | `/dev/sdb1` | `parted`, `fdisk` |
| Sistema de archivos | la estructura que organiza archivos y carpetas dentro de la partición | XFS, ext4 | `mkfs.xfs`, `mkfs.ext4` |
| Punto de montaje | la carpeta del árbol donde "aparece" ese sistema de archivos | `/archivos` | `mount`, `/etc/fstab` |

En Linux no hay `C:` ni `D:`. Todo cuelga de un solo árbol que empieza en `/`. Un disco nuevo aparece en la carpeta que vos elijas. Si lo desmontás, la carpeta queda vacía: los datos no están en la carpeta, están en el sistema de archivos.

## Los nombres en `/dev`

```bash
lsblk
```

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

| Nombre | Qué es |
|---|---|
| `sda`, `sdb`, `sdc` | los discos, en orden (VirtualBox) |
| `vda`, `vdb`, `vdc` | lo mismo en UTM |
| `sdb1`, `sdb2` | particiones del disco `sdb` |
| `rhel-root` | volumen LVM: grupo `rhel`, volumen `root` (también `/dev/mapper/rhel-root`) |
| `sr0` | el lector de CD |

`sdb` puede llamarse `sdc` en el próximo arranque si se agrega o quita un disco. Por eso en `/etc/fstab` no se usa el nombre: se usa el **UUID**.

## Lo que ya hizo el instalador

```bash
lsblk -f
```

- `sda1`: partición con XFS, montada en `/boot` (el kernel).
- `sda2`: **no tiene sistema de archivos**, dice `LVM2_member`: está entregada a LVM. Adentro hay dos volúmenes: `root` (`/`) y `swap`.
- `sdb` y `sdc`: los dos discos nuevos, vacíos. Con eso trabajamos hoy.
- En UTM hay una partición más, `vda1` de 600M en `/boot/efi`; el resto se corre un número.

## Tabla de particiones: GPT

| | MBR (`msdos`) | GPT |
|---|---|---|
| Año | 1983 | actual |
| Particiones | 4 | 128 |
| Disco máximo | 2 TiB | enorme |
| Hoy | solo por compatibilidad | **siempre** |

## Sistemas de archivos que importan

| | XFS | ext4 | vfat |
|---|---|---|---|
| Qué es | el de RHEL desde la versión 7 | el clásico de Linux | el de Windows y los pendrives |
| Crear | `mkfs.xfs` | `mkfs.ext4` | `mkfs.vfat` |
| Etiqueta | `xfs_admin -L` | `e2label` | `-n` al crear |
| Agrandar | sí, montado (`xfs_growfs`) | sí (`resize2fs`) | no |
| Achicar | **nunca** | sí, desmontado | no |
| Para qué | todo | compatibilidad | intercambio, `/boot/efi` |

**XFS no se achica.** Si te quedó grande: respaldar, destruir, crear más chico, restaurar. Por eso se empieza chico y se crece.

## UUID y etiqueta

```bash
sudo blkid
```

```
/dev/sda1: UUID="5d2f8b1a-..." TYPE="xfs" PARTUUID="..."
/dev/mapper/rhel-root: UUID="..." TYPE="xfs"
```

- **UUID**: número único que recibe cada sistema de archivos al crearse. No cambia aunque el disco pase de `sdb` a `sdc`. Es lo que va en `/etc/fstab`.
- **Etiqueta (label)**: un nombre que le ponés vos (`ARCHIVOS`). También sirve en `fstab`, como `LABEL=ARCHIVOS`.
- Si volvés a formatear (`mkfs`), el UUID **cambia**. Después de cada `mkfs`, mirar `blkid` de nuevo.
- `PARTUUID` es de la partición, no del sistema de archivos. No es el que va en `fstab`.

## Por qué LVM

Con particiones, el tamaño queda fijo. Si `/datos` se llena, hay que copiar todo a un disco más grande. LVM pone una capa en el medio:

```
 /dev/sdb3 ──► PV ──┐
                    ├──► VG vg_datos ──► LV lv_datos   ──► XFS  ──► /datos
 /dev/sdb4 ──► PV ──┘                   LV lv_backups ──► ext4 ──► /backups
```

| Sigla | Nombre | Qué es |
|---|---|---|
| PV | volumen físico | un disco o partición entregado a LVM |
| VG | grupo de volúmenes | la bolsa donde se juntan los PV; se ve como un solo espacio |
| LV | volumen lógico | las "particiones" que se recortan del VG; encima va el sistema de archivos |

Se agregan discos al VG cuando hace falta y se alarga el LV sin desmontar. El sistema ya usa LVM: `sda2` es un PV del VG `rhel`. El día que `/` se llene, se agrega un disco y se amplía igual que hoy.

## El mapa de hoy

| Disco | Partición | Tamaño | Para qué | Lab |
|---|---|---|---|---|
| `sdb` (con `parted`) | `sdb1` | 512 MiB | XFS → `/archivos` | 2.2, 2.3 |
| | `sdb2` | 512 MiB | swap | 2.4 |
| | `sdb3` | 2 GiB | PV → VG `vg_datos` → `/datos` | 3.1 |
| | `sdb4` | 2 GiB | PV → ampliar `vg_datos` | 3.2 |
| `sdc` (con `fdisk`) | `sdc1` | 3 GiB | Stratis `pool1` → `/stratis` | 4.1 |
| | `sdc2` | 2 GiB | ampliar `pool1` | 4.1 |
