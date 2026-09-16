# Día 06 — Almacenamiento: particiones, sistemas de archivos, montaje persistente, swap, LVM, Stratis y VDO
> Al terminar el día, el participante toma un disco nuevo, lo particiona, le crea un sistema de archivos, lo monta de forma persistente con `/etc/fstab`, añade swap, construye y amplía volúmenes LVM en caliente y crea un pool Stratis; además sabe qué es VDO y para qué sirve el automontaje.

**Ficha técnica cubierta:**
- RH124 M7: Particiones, Sistemas de archivos, Montaje de discos (completo).
- RH134 M6: LVM, Stratis, VDO. **Auto montaje:** hoy se explica el concepto (autofs y `x-systemd.automount`); la práctica se hace el **Día 9** junto con NFS, porque el caso de uso real del automontaje es montar recursos de red bajo demanda.

**Requisitos previos:**
- Snapshot `dia05-fin` tomado y VM `rhel01` arrancando bien (IP estática host-only del Día 5 funcionando o al menos acceso por `ssh -p 2222 student@localhost`).
- **Dos discos adicionales de 5 GB** añadidos a la VM (tarea del Día 5): `/dev/sdb` y `/dev/sdc` en VirtualBox; `/dev/vdb` y `/dev/vdc` en UTM. Se verifica en los primeros 10 minutos.
- VM registrada (`sudo subscription-manager status` → "Overall Status: Current") porque hoy se instalan paquetes: `stratis-cli`, `stratisd` y, solo en la VM del instructor, `vdo` (y `kmod-kvdo` únicamente si esa versión de RHEL 9 todavía lo necesita; ver Bloque 5).
- Contraseña de root a la mano: si alguien rompe `/etc/fstab`, se necesita para salir del *emergency mode*.

> **Aviso permanente del día:** todos los comandos están escritos con `/dev/sdb` y `/dev/sdc` (VirtualBox). Quien trabaje en UTM sustituye por `/dev/vdb` y `/dev/vdc`. Las particiones se nombran igual: `sdb1` ↔ `vdb1`. Todo lo demás (LVM, Stratis, fstab) es idéntico.

---

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---:|---|---|
| 0:00–0:10 | 10 | Repaso y verificación | Tres preguntas del Día 5; `lsblk` para confirmar los dos discos nuevos; si faltan, se añaden ahora |
| 0:10–0:35 | 25 | Bloque 1 — Conceptos | Disco → partición → sistema de archivos → punto de montaje; MBR vs GPT; XFS/ext4/vfat; UUID y labels; qué hizo el instalador; por qué LVM; mapa de discos del día |
| 0:35–1:30 | 55 | Bloque 2 — Particiones, FS, fstab y swap | Lab 2.1 particionar con parted y fdisk; Lab 2.2 mkfs, mount, labels; Lab 2.3 `/etc/fstab`; Lab 2.4 swap |
| 1:30–1:45 | 15 | Descanso | |
| 1:45–2:45 | 60 | Bloque 3 — LVM | Conceptos PV/VG/LV; Lab 3.1 crear y montar; Lab 3.2 extender en caliente; Lab 3.3 snapshot, renombrar y borrar |
| 2:45–3:10 | 25 | Bloque 4 — Stratis | Concepto; Lab 4.1 pool, filesystem, fstab, snapshot, add-data |
| 3:10–3:20 | 10 | Bloque 5 — VDO (demo del instructor) | Deduplicación y compresión con LVM-VDO; `vdostats` |
| 3:20–3:30 | 10 | Bloque 6 — Automontaje (concepto) | autofs (mapas maestro/indirecto/directo) y `x-systemd.automount`; referencia al Día 9 |
| 3:30–3:50 | 20 | Reto individual | Ticket: `lv_backups` en ext4 persistente + ampliación en caliente + archivo swap |
| 3:50–4:00 | 10 | Cierre | Estado final de los discos, cheatsheet, snapshot `dia06-fin`, tarea |

---

## Prioridad si falta tiempo

**Imprescindible** (con esto el participante sale sabiendo y practicando lo que pide el RHCSA):
- Verificar discos con `lsblk` y entender la cadena disco → partición → FS → punto de montaje.
- Particionar en GPT con `parted` (`mklabel`, `mkpart`, `print`, `rm`).
- `mkfs.xfs`, `mount`, `umount`, `df -h`, `findmnt`, `blkid`.
- `/etc/fstab` con `UUID=`, `defaults,nofail`, `systemctl daemon-reload`, `mount -a`. Saber qué hacer si la VM cae en *emergency mode* (caja de emergencia del Lab 2.3).
- Swap en partición: `mkswap`, `swapon`, `swapon --show`, línea en fstab.
- LVM: `pvcreate`, `vgcreate`, `lvcreate`, `mkfs.xfs`, montar persistente, `vgextend`, `lvextend -r`.

**Importante:**
- `fdisk` interactivo como alternativa a `parted`.
- `mkfs.ext4`, `e2label`, `xfs_admin -L`.
- "target is busy": `lsof`, `fuser`.
- Snapshot LVM, `lvrename`, `lvremove`, `-l 100%FREE`.
- Stratis completo (Lab 4.1).
- Reto individual (al menos la primera parte: crear `lv_backups` y el archivo swap).

**Si sobra tiempo** (puede quedar como demo del instructor o tarea):
- VDO (siempre es demo).
- Mini-lab de `x-systemd.automount` (Bloque 6).
- `mkfs.vfat` (paso 6b del Lab 2.2; requiere `dosfstools`), `/etc/lvm/lvm.conf` y el *devices file* de LVM.
- Reto opcional "/datos aparece lleno".

---

## Bloque 0 — Repaso y verificación de discos (10 min)

**Repaso (3 min), con la terminal abierta:**
1. ¿Con qué comando se ve el estado de todas las interfaces de red? (`nmcli device`)
2. ¿Qué pasa si se ejecuta `dnf install` en una VM no registrada? (no hay repositorios: "This system is not registered...")
3. ¿Dónde guarda NetworkManager en RHEL 9 los perfiles de conexión? (`/etc/NetworkManager/system-connections/*.nmconnection`)

**Verificación de los discos (7 min):**

```bash
lsblk
```

Salida esperada (VirtualBox):

```text
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

En UTM los nombres son `vda`, `vdb`, `vdc` y, por ser arranque UEFI, aparece además `vda1` de 600M montado en `/boot/efi` (entonces `/boot` es `vda2` y el LVM `vda3`). El contenido es el mismo. Si alguien creó la VM de VirtualBox con la casilla "Habilitar EFI" marcada verá lo mismo en `sda`: `sda1` (`/boot/efi`), `sda2` (`/boot`) y `sda3` (LVM). No es un error; en lo que sigue solo importan `sdb` y `sdc`.

**Qué observar:** `sdb` y `sdc` son de tipo `disk`, de 5G, **sin particiones ni punto de montaje**. Están vacíos: el sistema los ve pero no los usa.

**Si NO aparecen `sdb`/`sdc` (o `vdb`/`vdc`):** se añaden ahora, con la VM apagada (los controladores virtuales no admiten conexión en caliente de forma fiable).

```bash
sudo poweroff
```

- **VirtualBox:** Configuración → Almacenamiento → Controlador: SATA → icono "Agregar disco duro" → Crear → VDI → Reservado dinámicamente → 5 GB → Elegir. Repetir para el segundo disco.
- **UTM:** Editar VM → Unidades → Nuevo... → Interfaz VirtIO → Tamaño 5 GB → Guardar. Repetir.

Arrancar la VM, reconectar por SSH y repetir `lsblk`. Quien tenga problemas, restaura el snapshot `dia05-fin`, apaga y añade los discos.

**Checkpoint (todos pegan en el chat):**

```bash
lsblk -d -o NAME,SIZE,TYPE,MODEL
```

```text
NAME SIZE TYPE MODEL
sda   20G disk VBOX HARDDISK
sdb    5G disk VBOX HARDDISK
sdc    5G disk VBOX HARDDISK
sr0 1024M rom  CD-ROM
```

---

## Bloque 1 — Conceptos de almacenamiento (25 min)

### Conceptos (25 min)

**La cadena completa: disco → partición → sistema de archivos → punto de montaje.**

Qué decir en clase: "Un disco recién conectado es un bloque de bytes sin estructura. Para usarlo hay cuatro pasos y cada uno tiene su comando":

| Paso | Qué es | Analogía | Comando |
|---|---|---|---|
| Disco | El dispositivo físico o virtual (`/dev/sdb`) | Un terreno baldío | `lsblk` |
| Partición | Una porción del disco con inicio y fin (`/dev/sdb1`) | Dividir el terreno en lotes con cerca | `parted`, `fdisk` |
| Sistema de archivos | La estructura que organiza archivos y directorios dentro de la partición | Construir el edificio con sus pisos y oficinas | `mkfs.xfs`, `mkfs.ext4` |
| Punto de montaje | El directorio del árbol donde "aparece" ese sistema de archivos | Poner la dirección en la puerta para que la gente llegue | `mount`, `/etc/fstab` |

En Linux **no hay letras de unidad** (C:, D:). Todo se cuelga de un único árbol que empieza en `/`. Un disco nuevo "aparece" en el directorio que uno decida: `/datos`, `/backups`, `/srv/web`. Si se desmonta, el directorio sigue existiendo pero vacío: los datos no están en el directorio, están en el sistema de archivos.

**Dispositivos en `/dev`.** El kernel expone cada disco como un archivo de bloques. En VirtualBox (controlador SATA emulado) son `sda`, `sdb`, `sdc`; en UTM (controlador virtio, más eficiente) son `vda`, `vdb`, `vdc`. Las particiones añaden un número: `sdb1`, `sdb2`. Los volúmenes LVM aparecen como `/dev/mapper/vg-lv` (y el alias `/dev/vg/lv`). Los nombres `sdX` **pueden cambiar entre reinicios** si se agregan o quitan discos; por eso en `/etc/fstab` se usa el **UUID**.

**MBR vs GPT.**
- **MBR (msdos):** tabla de particiones de 1983. Máximo 4 particiones primarias (o 3 + una extendida con lógicas), discos hasta 2 TiB. Todavía se ve en servidores viejos.
- **GPT (GUID Partition Table):** el estándar actual. Hasta 128 particiones, discos de tamaño enorme, cada partición tiene un GUID único, tabla de respaldo al final del disco. **Hoy se usa GPT siempre**, salvo compatibilidad con algo antiguo.
- Dato práctico: en `parted`, con GPT, la palabra `primary` en `mkpart primary ...` no es un tipo (como en MBR) sino simplemente el **nombre** de la partición. Se sigue escribiendo por costumbre y porque así lo hacen los materiales oficiales de Red Hat.

**Sistemas de archivos que importan en RHEL 9.**
- **XFS:** el predeterminado de RHEL desde la versión 7. Muy rápido con archivos grandes, escala a petabytes, se puede **agrandar en caliente** (`xfs_growfs`) pero **NO se puede reducir**. Herramientas: `mkfs.xfs`, `xfs_admin`, `xfs_growfs`, `xfs_repair`. Decirlo tres veces en clase: **XFS no se reduce**. Si alguien pregunta "¿y si me equivoqué y lo hice muy grande?": se respalda, se destruye, se crea más chico y se restaura. Por eso se empieza pequeño y se crece.
- **ext4:** el clásico de Linux, sólido y muy compatible. Se puede agrandar y **también reducir** (desmontado). Herramientas: `mkfs.ext4`, `e2label`, `resize2fs`, `e2fsck`, `tune2fs`.
- **vfat (FAT32):** para intercambiar con Windows, memorias USB y la partición EFI (`/boot/efi`). Sin permisos Unix, sin archivos mayores de 4 GiB. `mkfs.vfat` viene en el paquete `dosfstools`.
- El RHCSA pide saber crear y montar los tres.

**UUID y labels.** Cada sistema de archivos recibe al crearse un **UUID** (identificador único de 128 bits, por ejemplo `3f0c2b7e-9a1d-4c55-8a2f-1b7e6d0c9f41`). El UUID no cambia aunque el disco cambie de `sdb` a `sdc`; por eso es lo que se pone en `/etc/fstab`. Un **label** (etiqueta) es un nombre legible que uno mismo asigna (`DATOS`, `BACKUPS`); también se puede usar en fstab con `LABEL=`. Se ven con `blkid` y `lsblk -f`. Ojo: si se vuelve a formatear (`mkfs`), el UUID cambia y la línea de fstab queda apuntando a algo que ya no existe.

**Qué hizo el instalador (leer el `lsblk` del inicio con los participantes):**
- `sda1` (1G, XFS) montada en `/boot`: kernel e initramfs.
- `sda2` (19G): no tiene sistema de archivos directamente; es un **volumen físico LVM** que forma el grupo de volúmenes `rhel`, dentro del cual hay dos volúmenes lógicos: `rhel-root` (17G, XFS, montado en `/`) y `rhel-swap` (2G).
- Es decir: **el sistema ya usa LVM**. Lo que se aprende hoy sirve también para agrandar `/` el día que se quede sin espacio (se añade un disco, `vgextend rhel`, `lvextend -r`).

**Por qué LVM (Logical Volume Manager).** Con particiones clásicas el tamaño queda fijo y contiguo: si `/datos` se llena, hay que copiar todo a un disco más grande. LVM pone una capa de abstracción:
- **PV** (Physical Volume): un disco o partición "entregado" a LVM.
- **VG** (Volume Group): la bolsa donde se juntan uno o más PV. Se ve como un solo espacio.
- **LV** (Logical Volume): las "particiones" que se recortan del VG. Sobre ellas va el sistema de archivos.
- **PE** (Physical Extent): la unidad mínima de asignación, 4 MiB por defecto.

Analogía: los PV son bolsas de ladrillos, el VG es la bodega donde se vacían todas las bolsas, los LV son las paredes que uno construye con esos ladrillos. Se pueden agregar bolsas (discos) a la bodega en cualquier momento y alargar una pared sin tumbarla. Eso es exactamente `vgextend` + `lvextend`.

**Mapa de discos del día (lo que se va a construir):**

```text
DISCO (VirtualBox / UTM)      PARTICIÓN     TAMAÑO     USO                                        LAB
------------------------------------------------------------------------------------------------------
/dev/sda  /  /dev/vda          sda1          1 GiB     XFS -> /boot            (lo hizo el instalador)
   20 GB, sistema              sda2         19 GiB     PV -> VG rhel -> LV root (/) y LV swap

/dev/sdb  /  /dev/vdb          sdb1        512 MiB     XFS  -> /archivos                          2.2, 2.3
   5 GB, GPT con parted        sdb2        512 MiB     swap                                       2.4
                               sdb3          2 GiB     PV   -> VG vg_datos                        3.1
                               sdb4         ~2 GiB     PV   -> vgextend vg_datos                  3.2
                                                       LV lv_datos   (2 GiB, XFS)  -> /datos      3.1, 3.2
                                                       LV lv_backups (1.5 GiB, ext4) -> /backups  Reto

/dev/sdc  /  /dev/vdc          sdc1          3 GiB     Stratis pool1 -> fs1 -> /stratis           4.1
   5 GB, GPT con fdisk         sdc2         ~2 GiB     stratis pool add-data pool1                4.1

/swapfile (en /)                            512 MiB    archivo swap persistente                   Reto
```

Decir en clase: "sdb lo particionamos con `parted` y sdc con `fdisk`, para que conozcan las dos herramientas. sdb es nuestro disco de LVM; sdc queda entero para Stratis, que necesita dispositivos sin nada encima."

---

## Bloque 2 — Particiones, sistemas de archivos, fstab y swap (55 min)

### Lab 2.1 — Particionar sdb con parted y sdc con fdisk (18 min)

- **Objetivo:** crear en `/dev/sdb` una tabla GPT con cuatro particiones usando `parted`, y en `/dev/sdc` dos particiones usando `fdisk`, verificando con `lsblk` y `fdisk -l`.

1. Reconocer los discos y confirmar que están vacíos (sin tabla de particiones ni firma de sistema de archivos):

```bash
lsblk -f /dev/sdb /dev/sdc
sudo blkid /dev/sdb /dev/sdc
sudo fdisk -l /dev/sdb
```

Salida esperada:

```text
NAME FSTYPE FSVER LABEL UUID FSAVAIL FSUSE% MOUNTPOINTS
sdb
sdc
```

`blkid` no imprime nada (no hay firma). `fdisk -l /dev/sdb` muestra:

```text
Disk /dev/sdb: 5 GiB, 5368709120 bytes, 10485760 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

**Qué observar:** no hay línea "Disklabel type": el disco no tiene tabla de particiones. En UTM el modelo dice `Virtio Block Device`.

2. Abrir `parted` sobre sdb en modo interactivo (el prompt cambia a `(parted)`):

```bash
sudo parted /dev/sdb
```

```text
GNU Parted 3.5
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)
```

3. Ver el estado, crear la tabla GPT y fijar la unidad en MiB:

```text
(parted) print
Error: /dev/sdb: unrecognised disk label
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags:
(parted) mklabel gpt
(parted) unit MiB
(parted) print
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 5120MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start  End  Size  File system  Name  Flags
```

**Qué observar:** `mklabel gpt` se aplica **inmediatamente** (parted no tiene "deshacer"; fdisk sí, porque escribe solo con `w`). `unit MiB` hace que todo se muestre y se interprete en MiB.

4. Crear la partición para XFS y la de swap:

```text
(parted) mkpart primary xfs 1MiB 513MiB
(parted) mkpart primary linux-swap 513MiB 1025MiB
```

Se empieza en 1MiB (no en 0) para dejar espacio a la tabla GPT y quedar alineado; si parted dice `Warning: The resulting partition is not properly aligned for best performance`, se responde `Ignore` en el lab, pero en producción se corrigen los límites a múltiplos de 1MiB.

5. Equivocarse a propósito y corregir con `rm`: crear la tercera partición muy pequeña, verla, borrarla y volverla a crear bien.

```text
(parted) mkpart primary 1025MiB 2049MiB
(parted) print
Number  Start    End      Size     File system     Name     Flags
 1      1.00MiB  513MiB   512MiB   xfs             primary
 2      513MiB   1025MiB  512MiB   linux-swap(v1)  primary  swap
 3      1025MiB  2049MiB  1024MiB                  primary
(parted) rm 3
(parted) mkpart primary 1025MiB 3073MiB
(parted) mkpart primary 3073MiB 100%
(parted) set 3 lvm on
(parted) set 4 lvm on
(parted) print
Number  Start    End      Size     File system     Name     Flags
 1      1.00MiB  513MiB   512MiB   xfs             primary
 2      513MiB   1025MiB  512MiB   linux-swap(v1)  primary  swap
 3      1025MiB  3073MiB  2048MiB                  primary  lvm
 4      3073MiB  5119MiB  2046MiB                  primary  lvm
(parted) quit
Information: You may need to update /etc/fstab.
```

(El fin exacto de la partición 4 y su tamaño, `5119MiB`/`2046MiB`, pueden variar uno o dos MiB según cómo parted alinee el final del disco: es normal y no hay que corregirlo.)

**Qué observar:** la columna "File system" de parted solo refleja la **intención** declarada en `mkpart` (todavía no hay ningún sistema de archivos real: eso lo hace `mkfs`). `set N lvm on` marca el tipo de partición como "Linux LVM"; no es obligatorio para que LVM funcione, pero documenta el propósito y es lo que espera cualquier administrador que mire el disco después. `100%` significa "hasta el final del disco".

6. Pedir al kernel que relea la tabla y verificar:

```bash
sudo udevadm settle
lsblk /dev/sdb
sudo fdisk -l /dev/sdb
```

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0    5G  0 disk
├─sdb1   8:17   0  512M  0 part
├─sdb2   8:18   0  512M  0 part
├─sdb3   8:19   0    2G  0 part
└─sdb4   8:20   0    2G  0 part
```

```text
Disklabel type: gpt
Disk identifier: 6E7A1C2B-...

Device       Start      End Sectors  Size Type
/dev/sdb1     2048  1050623 1048576  512M Linux filesystem
/dev/sdb2  1050624  2099199 1048576  512M Linux swap
/dev/sdb3  2099200  6293503 4194304    2G Linux LVM
/dev/sdb4  6293504 10483711 4190208    2G Linux LVM
```

(Los sectores de inicio y fin de `sdb4` pueden diferir ligeramente de los mostrados; lo importante es que haya cuatro particiones con esos tamaños y tipos.)

Normalmente el kernel se entera solo; si `lsblk` no muestra las particiones, `sudo partprobe /dev/sdb` fuerza la relectura (solo funciona si ninguna partición del disco está en uso) y `udevadm settle` espera a que udev termine de crear los nodos en `/dev`.

7. Ahora **sdc con `fdisk`** (la alternativa clásica; escribe en disco solo al final con `w`):

```bash
sudo fdisk /dev/sdc
```

Secuencia de teclas y respuestas (lo que va después de `:` lo escribe el participante):

```text
Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x2f9a1c3e.

Command (m for help): g
Created a new GPT disklabel (GUID: 1D4E...).

Command (m for help): n
Partition number (1-128, default 1): [Enter]
First sector (2048-10485726, default 2048): [Enter]
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-10485726, default 10485726): +3G

Created a new partition 1 of type 'Linux filesystem' and of size 3 GiB.

Command (m for help): n
Partition number (2-128, default 2): [Enter]
First sector (6293504-10485726, default 6293504): [Enter]
Last sector, +/-sectors or +/-size{K,M,G,T,P} (6293504-10485726, default 10485726): [Enter]

Created a new partition 2 of type 'Linux filesystem' and of size 2 GiB.

Command (m for help): p
Disk /dev/sdc: 5 GiB, 5368709120 bytes, 10485760 sectors
...
Device       Start      End Sectors Size Type
/dev/sdc1     2048  6293503 6291456   3G Linux filesystem
/dev/sdc2  6293504 10485726 4192223   2G Linux filesystem

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Teclas de fdisk que hay que memorizar: `g` (nueva tabla GPT), `o` (nueva tabla MBR), `n` (nueva partición), `d` (borrar), `t` (cambiar tipo: por ejemplo `swap` o `lvm`, o `L` para listar), `p` (imprimir), `q` (salir **sin** guardar), `w` (escribir y salir). Para Stratis dejamos el tipo "Linux filesystem", así que hoy no usamos `t` en sdc; en sdb habríamos hecho `t` → `2` → `swap` y `t` → `3` → `lvm`.

```bash
sudo udevadm settle
lsblk /dev/sdc
```

```text
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sdc      8:32   0   5G  0 disk
├─sdc1   8:33   0   3G  0 part
└─sdc2   8:34   0   2G  0 part
```

- **Checkpoint:**

```bash
lsblk -o NAME,SIZE,TYPE,PARTTYPENAME /dev/sdb /dev/sdc
```

```text
NAME   SIZE TYPE PARTTYPENAME
sdb      5G disk
├─sdb1 512M part Linux filesystem
├─sdb2 512M part Linux swap
├─sdb3   2G part Linux LVM
└─sdb4   2G part Linux LVM
sdc      5G disk
├─sdc1   3G part Linux filesystem
└─sdc2   2G part Linux filesystem
```

(⚠️ Verificar en la VM antes de la clase que la columna `PARTTYPENAME` existe en la versión de `lsblk` instalada. Si respondiera `lsblk: unknown column: PARTTYPENAME`, el checkpoint equivalente es `sudo fdisk -l /dev/sdb /dev/sdc | grep -E "^/dev"`.)

### Lab 2.2 — Sistemas de archivos, montaje manual y labels (12 min)

- **Objetivo:** formatear `sdb1` en XFS, etiquetarlo, montarlo y desmontarlo, probar `ro`, ver qué hace `mkfs.ext4`, y volver a XFS entendiendo que cada `mkfs` destruye lo anterior y cambia el UUID.

1. Crear el sistema de archivos XFS en `sdb1`:

```bash
sudo mkfs.xfs /dev/sdb1
```

```text
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=32768 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=131072, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

**Qué observar:** `blocks=131072` × `bsize=4096` = 512 MiB. El log interno se lleva 2560 bloques (10 MiB): ese es el mínimo que reserva `mkfs.xfs` aunque el sistema de archivos sea pequeño, y por eso `df` mostrará menos de 512M. Tardó un segundo: `mkfs` no borra los datos byte a byte, solo escribe la estructura (metadatos).

2. Ver la firma que acaba de aparecer y ponerle un label (XFS: máximo 12 caracteres, con el FS desmontado):

```bash
sudo blkid /dev/sdb1
sudo xfs_admin -L ARCHIVOS /dev/sdb1
sudo blkid /dev/sdb1
```

```text
/dev/sdb1: UUID="3f0c2b7e-9a1d-4c55-8a2f-1b7e6d0c9f41" TYPE="xfs" PARTLABEL="primary" PARTUUID="a1b2c3d4-..."
writing all SBs
new label = "ARCHIVOS"
/dev/sdb1: LABEL="ARCHIVOS" UUID="3f0c2b7e-9a1d-4c55-8a2f-1b7e6d0c9f41" TYPE="xfs" PARTLABEL="primary" PARTUUID="a1b2c3d4-..."
```

**Qué observar:** `UUID` es del sistema de archivos (lo usa fstab); `PARTUUID` y `PARTLABEL` son de la partición GPT (los puso parted). No confundirlos.

3. Crear el punto de montaje, montar y comprobar:

```bash
sudo mkdir /archivos
sudo mount /dev/sdb1 /archivos
findmnt /archivos
df -h /archivos
```

```text
TARGET    SOURCE    FSTYPE OPTIONS
/archivos /dev/sdb1 xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       502M   29M  473M   6% /archivos
```

**Qué observar:** `seclabel` indica que SELinux etiqueta los archivos de este FS. El tamaño útil (unos `502M`; puede variar unos MiB según la versión de xfsprogs) es menor que 512M porque el log interno y los metadatos de los grupos de asignación ocupan espacio.

4. Demostrar que los datos viven en el FS, no en el directorio:

```bash
sudo touch /archivos/prueba1.txt
echo "hola desde sdb1" | sudo tee /archivos/nota.txt
ls -l /archivos
sudo umount /archivos
ls -l /archivos
sudo mount /dev/sdb1 /archivos
ls -l /archivos
```

Tras `umount`, `ls` muestra el directorio **vacío**; tras volver a montar, los archivos reaparecen. Si alguien escribe archivos en `/archivos` mientras está desmontado, esos archivos quedan "tapados" cuando se monta encima: es un error clásico que llena el disco raíz sin que `du` lo explique.

5. Montaje de solo lectura (útil para inspeccionar un disco sospechoso sin alterarlo). *Si el bloque va atrasado, este paso lo demuestra el instructor y los participantes pasan al 6:*

```bash
sudo umount /archivos
sudo mount -o ro /dev/sdb1 /archivos
sudo touch /archivos/nuevo.txt
sudo mount -o remount,rw /archivos
sudo touch /archivos/nuevo.txt && echo OK
```

```text
touch: cannot touch '/archivos/nuevo.txt': Read-only file system
OK
```

`remount` cambia opciones sin desmontar. Se usa también en el sentido contrario para congelar un FS con errores.

6. ¿Y si se quiere ext4? Se formatea de nuevo (destruye todo) y se etiqueta con `e2label`:

```bash
sudo umount /archivos
sudo mkfs.ext4 /dev/sdb1
```

```text
mke2fs 1.46.5 (30-Dec-2021)
/dev/sdb1 contains a xfs file system labelled 'ARCHIVOS'
Proceed anyway? (y,N) y
Creating filesystem with 131072 4k blocks and 32768 inodes
Filesystem UUID: 7b9d4e10-2c3f-4a8b-9e6d-5f1a2b3c4d5e
Superblock backups stored on blocks:
        32768, 98304
Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

```bash
sudo e2label /dev/sdb1 ARCHIVOS
sudo blkid /dev/sdb1
```

```text
/dev/sdb1: LABEL="ARCHIVOS" UUID="7b9d4e10-2c3f-4a8b-9e6d-5f1a2b3c4d5e" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="primary" PARTUUID="a1b2c3d4-..."
```

**Qué observar:** `mkfs.ext4` **preguntó** porque detectó una firma XFS; el **UUID cambió**. Si `/etc/fstab` ya tuviera una línea con el UUID viejo, el próximo arranque fallaría. Regla: después de cualquier `mkfs`, volver a mirar `blkid`.

6b. **(Si sobra tiempo, o demo del instructor en 2 min.)** El RHCSA pide también **vfat**. `mkfs.vfat` viene en el paquete `dosfstools`, que no está instalado por defecto:

```bash
sudo dnf install -y dosfstools
sudo wipefs -a /dev/sdb1              # quita la firma ext4 anterior: mkfs.vfat no avisa de lo que sobrescribe
sudo mkfs.vfat -n DATOSFAT /dev/sdb1
sudo mount /dev/sdb1 /archivos
df -hT /archivos
sudo umount /archivos
```

```text
mkfs.fat 4.2 (2021-01-31)
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      vfat  511M     0  511M   0% /archivos
```

**Qué observar:** el label vfat va en mayúsculas y admite máximo 11 caracteres. `ls -l` dentro de un vfat muestra todos los archivos con los mismos permisos: FAT32 **no guarda permisos ni propietario** de Unix, los inventa a partir de las opciones de montaje (`uid=`, `gid=`, `umask=`). Por eso no sirve para `/home` ni para datos de servidor, solo para intercambio con Windows y para `/boot/efi`.

7. Volver a XFS (es el predeterminado de RHEL y el que usaremos en fstab). `mkfs.xfs` no pregunta: se niega, y hay que forzar con `-f`:

```bash
sudo mkfs.xfs /dev/sdb1
sudo mkfs.xfs -f -L ARCHIVOS /dev/sdb1
sudo blkid /dev/sdb1
```

```text
mkfs.xfs: /dev/sdb1 appears to contain an existing filesystem (ext4).
mkfs.xfs: Use the -f option to force overwrite.
...
/dev/sdb1: LABEL="ARCHIVOS" UUID="c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90" TYPE="xfs" ...
```

(Si se hizo el paso 6b, el mensaje dirá `(vfat)` en vez de `(ext4)`.) (`-L` pone el label al crear, equivalente a `xfs_admin -L` después.) Otro UUID nuevo: este es el que irá a fstab.

- **Checkpoint:**

```bash
sudo blkid -s UUID -s TYPE -s LABEL /dev/sdb1
```

```text
/dev/sdb1: LABEL="ARCHIVOS" UUID="c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90" TYPE="xfs"
```

(El UUID de cada participante es distinto; lo importante es `TYPE="xfs"` y el label.)

### Lab 2.3 — Montaje persistente con /etc/fstab (13 min)

- **Objetivo:** dejar `/archivos` montado en cada arranque usando UUID, entender cada campo de fstab, recargar systemd, probar con `mount -a` y saber resolver "target is busy" y el *emergency mode*.

1. Leer el fstab que dejó el instalador:

```bash
cat /etc/fstab
```

```text
#
# /etc/fstab
# Created by anaconda on Mon Aug 24 14:02:11 2026
#
# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
#
# After editing this file, run 'systemctl daemon-reload' to update systemd
# units generated from this file.
#
/dev/mapper/rhel-root   /                       xfs     defaults        0 0
UUID=5d2f8b1a-6c4e-4f3a-b2d1-8e9f0a1b2c3d /boot                   xfs     defaults        0 0
/dev/mapper/rhel-swap   none                    swap    defaults        0 0
```

**Los seis campos**, explicados sobre la línea de `/boot`:

| # | Campo | Ejemplo | Significado |
|---|---|---|---|
| 1 | Dispositivo | `UUID=5d2f...` | Qué montar. Puede ser `UUID=`, `LABEL=`, `/dev/mapper/vg-lv` (nombres LVM son estables) o `/dev/sdb1` (**evitar**: cambia) |
| 2 | Punto de montaje | `/boot` | Dónde. El directorio debe existir. `none` o `swap` para swap |
| 3 | Tipo | `xfs` | `xfs`, `ext4`, `vfat`, `swap`, `nfs`, `auto` |
| 4 | Opciones | `defaults` | Separadas por coma: `defaults` (= rw,suid,dev,exec,auto,nouser,async), `ro`, `noexec`, `nofail`, `noauto`, `x-systemd.*` |
| 5 | dump | `0` | Herencia de la utilidad `dump`; siempre 0 |
| 6 | pass | `0` | Orden de `fsck` al arrancar: 1 para `/`, 2 para el resto con ext4, **0 para XFS** (no usa fsck al arrancar) y para swap |

Fíjense en el comentario del propio archivo: Red Hat ya avisa que hay que ejecutar `systemctl daemon-reload` después de editarlo, porque systemd convierte cada línea en una unidad `.mount`.

2. Añadir la línea de `/archivos`. Se puede editar con `sudo vi /etc/fstab` (así se hace en el examen), pero para que **nadie se equivoque copiando el UUID** se usa este comando que lo lee con `blkid` y lo agrega al final:

```bash
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb1)  /archivos  xfs  defaults,nofail  0 0" | sudo tee -a /etc/fstab
tail -1 /etc/fstab
```

```text
UUID=c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90  /archivos  xfs  defaults,nofail  0 0
```

`nofail`: si el disco no está o falla, el arranque **continúa** sin ese punto de montaje. Sin `nofail`, systemd espera 90 segundos al dispositivo y luego cae al *emergency mode*. Para discos de datos secundarios, `nofail` es la opción segura; para `/` y `/boot` no se pone.

3. Recargar systemd y probar **antes de reiniciar**. `mount -a` monta todo lo que está en fstab y aún no está montado: si hay un error en la línea, aparece ahora y no en el próximo arranque.

```bash
findmnt /archivos || echo "no está montado: correcto, quedó desmontado al final del Lab 2.2"
sudo systemctl daemon-reload
sudo mount -a
findmnt /archivos
sudo findmnt --verify
```

```text
no está montado: correcto, quedó desmontado al final del Lab 2.2
TARGET    SOURCE    FSTYPE OPTIONS
/archivos /dev/sdb1 xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
Success, no errors or warnings detected
```

**Qué observar:** nadie montó `/archivos` a mano: lo montó `mount -a` leyendo la línea nueva de `/etc/fstab`. Eso es exactamente lo que hará systemd en el próximo arranque.

Si se olvida el `daemon-reload`, RHEL 9 lo recuerda al hacer `mount`:

```text
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
```

`findmnt --verify` revisa la sintaxis de fstab, la existencia de los UUID y de los directorios: correrlo siempre antes de un `reboot`.

4. Provocar y resolver "target is busy":

```bash
cd /archivos
sudo umount /archivos
```

```text
umount: /archivos: target is busy.
```

Algún proceso tiene un archivo abierto o su directorio de trabajo dentro del FS. Localizarlo:

```bash
sudo fuser -vm /archivos
sudo lsof /archivos
```

```text
                     USER        PID ACCESS COMMAND
/archivos:           student    2417 ..c.. bash
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
bash    2417 student  cwd    DIR   8,17       37  128 /archivos
```

`c` en ACCESS = *current directory*. (`lsof` viene en la instalación tipo Server; si faltara: `sudo dnf install -y lsof` o usar solo `fuser`, que está en `psmisc`.) Solución: salir del directorio (o cerrar el programa) y desmontar:

```bash
cd ~
sudo umount /archivos
sudo mount -a
df -h /archivos
```

Opciones de último recurso, en orden de agresividad: `umount -l` (lazy: desmonta cuando se libere), `fuser -km /archivos` (**mata** los procesos que lo usan; no se usa sin pensar).

5. **El peligro de un fstab malo.** No lo vamos a ejecutar, pero hay que saberlo: si se escribe mal el UUID o el punto de montaje **sin `nofail`**, la VM arranca así:

```text
[  TIME ] Timed out waiting for device dev-disk-by\x2duuid-....device
[DEPEND] Dependency failed for /datos.
[DEPEND] Dependency failed for Local File Systems.
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```

> **Caja de emergencia — cómo salir del emergency mode (se profundiza el Día 10):**
> 1. Hay que estar en la **consola de la VM** (ventana de VirtualBox/UTM): SSH no funciona porque la red no llegó a levantarse. Escribir la contraseña de **root**.
> 2. `journalctl -xb | grep -iE "mount|fstab|Dependency" | tail -20` o `systemctl --failed` para identificar la unidad que falló (por ejemplo `datos.mount`).
> 3. Asegurar que `/` está en escritura: `mount -o remount,rw /` (si ya lo estaba, no hace daño).
> 4. `vi /etc/fstab` y corregir la línea (o comentarla con `#` al inicio).
> 5. `systemctl daemon-reload && mount -a`. Si no da error: `systemctl default` (continúa el arranque) o `reboot`.
>
> Regla de oro: **`mount -a` y `findmnt --verify` antes de cada reinicio**, y `nofail` en discos de datos.

- **Checkpoint:**

```bash
grep archivos /etc/fstab && findmnt -n -o TARGET,SOURCE,FSTYPE /archivos
```

```text
UUID=c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90  /archivos  xfs  defaults,nofail  0 0
/archivos /dev/sdb1 xfs
```

### Lab 2.4 — Swap en partición (12 min)

- **Objetivo:** activar `sdb2` como swap, verla con `swapon --show` y `free -m`, hacerla persistente y conocer la alternativa de archivo swap y las prioridades.

Qué decir en clase: "La swap es espacio en disco que el kernel usa como extensión de la RAM cuando se agota. No hace más rápido al servidor; evita que se caiga. Una regla práctica: con menos de 2 GB de RAM, swap = 2×RAM; entre 2 y 8 GB, swap = RAM; más de 8 GB, 4–8 GB (o lo que pida la aplicación). La VM trae 2 GB de swap en `rhel-swap`."

1. Estado actual:

```bash
swapon --show
free -m
```

```text
NAME      TYPE      SIZE USED PRIO
/dev/dm-1 partition   2G   0B   -2
               total        used        free      shared  buff/cache   available
Mem:            3663         318        3081           8         263        3128
Swap:           2047           0        2047
```

2. Formatear `sdb2` como swap (equivalente al `mkfs` de la swap) y activarla:

```bash
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
free -m | grep -i swap
```

```text
Setting up swapspace version 1, size = 512 MiB (536866816 bytes)
no label, UUID=9e1f2a3b-4c5d-6e7f-8a9b-0c1d2e3f4a5b
NAME      TYPE      SIZE USED PRIO
/dev/dm-1 partition   2G   0B   -2
/dev/sdb2 partition 512M   0B   -3
Swap:           2559           0        2559
```

**Qué observar:** `PRIO` negativa asignada automáticamente: el kernel usa primero la de mayor prioridad (-2 antes que -3). Con `swapon -p 10 /dev/sdb2` (o la opción `pri=10` en fstab) se puede preferir una swap que esté en un disco más rápido.

3. Hacerla persistente. Campo 2 y 3 son `swap` (el instalador usa `none` en el campo 2; ambos valen):

```bash
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb2)  swap  swap  defaults  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
```

4. Probar sin reiniciar: desactivar y volver a activar **desde fstab** con `swapon -a`:

```bash
sudo swapoff /dev/sdb2
swapon --show
sudo swapon -a
swapon --show
```

Tras `swapoff` desaparece `/dev/sdb2`; tras `swapon -a` vuelve, lo que demuestra que la línea de fstab es correcta.

5. Alternativa: **archivo swap** (cuando no hay partición libre; se practica en el reto). Solo para verlo ahora:

```text
sudo dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress   # crea y escribe el archivo completo
sudo chmod 600 /swapfile                                             # obligatorio: mkswap avisa si es legible por otros
sudo restorecon -v /swapfile                                         # etiqueta SELinux correcta (swapfile_t)
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile  swap  swap  defaults  0 0" | sudo tee -a /etc/fstab
```

`dd` es el método que documenta Red Hat y el que hay que usar en el examen y en producción: escribe el archivo entero, así que nunca queda "con huecos". **No usar `fallocate` para swap:** sobre XFS (que es lo que hay en `/`) crea extents no escritos y `swapon` puede rechazar el archivo con `swapon: /swapfile: skipping - it appears to have holes`. En el reto se usa `dd`.

- **Checkpoint:**

```bash
swapon --show --noheadings | wc -l ; grep -c swap /etc/fstab
```

```text
2
2
```

(Dos swaps activas y dos líneas con "swap" en fstab.)

---

## Descanso (15 min)

---

## Bloque 3 — LVM (60 min)

### Conceptos (10 min)

Retomar la analogía de los ladrillos con el diagrama en pantalla:

```text
 Particiones / discos      Volúmenes físicos      Grupo de volúmenes          Volúmenes lógicos        Sistema de archivos
 /dev/sdb3 (2 GiB)  --->   PV /dev/sdb3   --\                                 LV lv_datos   (2 GiB) --> XFS  -> /datos
                                             >--->  VG vg_datos (3.99 GiB) >
 /dev/sdb4 (2 GiB)  --->   PV /dev/sdb4   --/     (1022 PE de 4 MiB)         LV lv_backups (1.5 GiB) -> ext4 -> /backups
```

- **PV:** `pvcreate` escribe una cabecera LVM en el dispositivo. Desde ahí, ese dispositivo "pertenece" a LVM y `blkid` lo muestra como `LVM2_member`.
- **VG:** `vgcreate nombre PV...`. Es la unidad de administración. Un VG puede crecer en cualquier momento con `vgextend`.
- **LV:** `lvcreate -n nombre -L tamaño VG`. Aparece como `/dev/VG/LV` y `/dev/mapper/VG-LV` (dos enlaces al mismo `/dev/dm-N`). Se formatea y monta como cualquier partición.
- **PE:** 4 MiB. Los tamaños se redondean al PE: pedir `-L 500M` da 125 PE = 500 MiB, pero pedir `-L 1G` en un VG que tiene "casi 1G" falla por un puñado de extents. Por eso existe `-l` (ele minúscula): `-l 100%FREE`, `-l 50%VG`, `-l 256` (extents).
- **Comandos de consulta:** `pvs`/`vgs`/`lvs` (resumen, una línea por objeto) y `pvdisplay`/`vgdisplay`/`lvdisplay` (detalle). Se recuerdan fácil: la letra del objeto + `s` o `display`.
- **Ampliar:** `vgextend` (más ladrillos a la bodega) y `lvextend -r` (alargar la pared **y** redimensionar el sistema de archivos en el mismo comando: `-r` = `--resizefs`, que llama a `xfs_growfs` o `resize2fs`). **Olvidar `-r` es el error número uno**: el LV crece, pero `df -h` sigue mostrando el tamaño viejo.
- **Reducir:** solo ext4, desmontado y con respaldo (`lvreduce -r -L -500M ...`). XFS no se reduce. **No lo practicamos.**
- **Orden de destrucción:** `umount` → `lvremove` → `vgremove` → `pvremove`. Al revés no funciona.
- En RHEL 9 LVM mantiene una lista de dispositivos permitidos en `/etc/lvm/devices/system.devices` (*devices file*); `pvcreate` añade el dispositivo automáticamente. Si un disco traído de otro servidor no aparece en `pvs`, se agrega con `lvmdevices --adddev /dev/sdX`. Las opciones globales (filtros, autoextend de snapshots, etc.) están en `/etc/lvm/lvm.conf`: hoy solo hay que saber que existe.

### Lab 3.1 — Crear PV, VG y LV, formatear y montar persistente (20 min)

- **Objetivo:** construir `vg_datos` sobre `sdb3`, crear `lv_datos` de 1 GiB con XFS y dejarlo montado en `/datos` de forma persistente.

1. Volumen físico:

```bash
sudo pvcreate /dev/sdb3
sudo pvs
sudo pvdisplay /dev/sdb3
```

```text
  Physical volume "/dev/sdb3" successfully created.
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda2  rhel lvm2 a--  <19.00g     0
  /dev/sdb3       lvm2 ---    2.00g  2.00g
```

```text
  "/dev/sdb3" is a new physical volume of "2.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb3
  VG Name
  PV Size               2.00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               Xk3r0p-...
```

**Qué observar:** `/dev/sda2` ya era PV del VG `rhel` (lo hizo el instalador). El PV nuevo no tiene VG todavía y por eso "PE Size 0": el tamaño de extent lo define el VG.

2. Grupo de volúmenes:

```bash
sudo vgcreate vg_datos /dev/sdb3
sudo vgs
sudo vgdisplay vg_datos
```

```text
  Volume group "vg_datos" successfully created
  VG       #PV #LV #SN Attr   VSize   VFree
  rhel       1   2   0 wz--n- <19.00g     0
  vg_datos   1   0   0 wz--n-  <2.00g <2.00g
```

```text
  --- Volume group ---
  VG Name               vg_datos
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <2.00 GiB
  PE Size               4.00 MiB
  Total PE              511
  Alloc PE / Size       0 / 0
  Free  PE / Size       511 / <2.00 GiB
  VG UUID               p7Qa2c-...
```

**Qué observar:** la partición mide 2048 MiB, pero LVM reserva el primer MiB para su metadata y luego reparte el resto en extents completos de 4 MiB: 511 PE × 4 MiB = 2044 MiB. Por eso `<2.00g` (el `<` de LVM significa "un poco menos de 2 GiB") y por eso `lvcreate -L 2G` aquí fallaría por unos pocos extents.

3. Volumen lógico de 1 GiB:

```bash
sudo lvcreate -n lv_datos -L 1G vg_datos
sudo lvs
sudo lvdisplay /dev/vg_datos/lv_datos
ls -l /dev/vg_datos/lv_datos /dev/mapper/vg_datos-lv_datos
```

```text
  Logical volume "lv_datos" created.
  LV       VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root     rhel     -wi-ao---- <17.00g
  swap     rhel     -wi-ao----   2.00g
  lv_datos vg_datos -wi-a-----   1.00g
```

```text
  --- Logical volume ---
  LV Path                /dev/vg_datos/lv_datos
  LV Name                lv_datos
  VG Name                vg_datos
  LV UUID                r2Dk8s-...
  LV Write Access        read/write
  LV Creation host, time rhel01, 2026-09-03 10:22:41 -0500
  LV Status              available
  # open                 0
  LV Size                1.00 GiB
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
```

```text
lrwxrwxrwx. 1 root root 7 Sep  3 10:22 /dev/mapper/vg_datos-lv_datos -> ../dm-2
lrwxrwxrwx. 1 root root 7 Sep  3 10:22 /dev/vg_datos/lv_datos -> ../dm-2
```

**Qué observar:** en `Attr`, `-wi-a-----`: `w` escribible, `i` asignación *inherited*, `a` activo; en `root` aparece además `o` (*open*: está montado). "Current LE 256" = 256 extents × 4 MiB = 1 GiB. Los dos nombres apuntan al mismo `dm-2`.

4. Formatear, montar y hacer persistente. Con LVM se puede usar el UUID **o** el nombre `/dev/mapper/vg_datos-lv_datos`, porque los nombres LVM son estables (no dependen de sdb/sdc). Usaremos el nombre de mapper para que se vea la diferencia con el Lab 2.3:

```bash
sudo mkfs.xfs /dev/vg_datos/lv_datos
sudo mkdir /datos
echo "/dev/mapper/vg_datos-lv_datos  /datos  xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /datos
```

```text
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_datos-lv_datos 1014M   40M  975M   4% /datos
```

(Sobre 1 GiB de LV, XFS deja unos 1014M útiles: el resto son el log interno y los metadatos. La cifra puede variar unos MiB según la versión de xfsprogs.)

5. Poner datos de prueba (reutilizamos la estructura `~/empresa` de días anteriores):

```bash
sudo mkdir -p /datos/empresa/{documentos,clientes,logs}
for i in 1 2 3 4 5; do echo "Contrato $i - PanamaTech" | sudo tee /datos/empresa/clientes/contrato_$i.txt > /dev/null; done
sudo dd if=/dev/zero of=/datos/empresa/logs/app.log bs=1M count=100 status=none
sudo du -sh /datos/empresa
df -h /datos
```

```text
101M    /datos/empresa
/dev/mapper/vg_datos-lv_datos 1014M  141M  874M  14% /datos
```

- **Checkpoint:**

```bash
sudo lvs vg_datos ; findmnt -n -o TARGET,SOURCE,FSTYPE /datos
```

```text
  LV       VG       Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_datos vg_datos -wi-ao---- 1.00g
/datos /dev/mapper/vg_datos-lv_datos xfs
```

### Lab 3.2 — Extender el VG y el LV en caliente (18 min)

- **Objetivo:** añadir `sdb4` al VG, ampliar `lv_datos` primero **sin** `-r` (para ver el error clásico y corregirlo con `xfs_growfs`) y después **con** `-r`, todo con `/datos` montado y en uso.

Escenario: "El equipo dice que `/datos` se va a quedar sin espacio. Hay que darle 1 GiB más sin desmontar y sin ventana de mantenimiento."

1. Extender el grupo con la partición reservada:

```bash
sudo pvcreate /dev/sdb4
sudo vgextend vg_datos /dev/sdb4
sudo vgs vg_datos
sudo pvs
```

```text
  Physical volume "/dev/sdb4" successfully created.
  Volume group "vg_datos" successfully extended
  VG       #PV #LV #SN Attr   VSize VFree
  vg_datos   2   1   0 wz--n- 3.99g 2.99g
  PV         VG       Fmt  Attr PSize   PFree
  /dev/sda2  rhel     lvm2 a--  <19.00g     0
  /dev/sdb3  vg_datos lvm2 a--   <2.00g 1020.00m
  /dev/sdb4  vg_datos lvm2 a--   <2.00g   <2.00g
```

(`vgextend` hace el `pvcreate` implícitamente si el dispositivo no es PV; se muestra por separado para que quede claro el paso.) Si fuera un disco entero se haría igual: `vgextend vg_datos /dev/sdd`.

2. Extender el LV **olvidando `-r`** a propósito:

```bash
sudo lvextend -L +512M /dev/vg_datos/lv_datos
sudo lvs vg_datos
df -h /datos
```

```text
  Size of logical volume vg_datos/lv_datos changed from 1.00 GiB (256 extents) to 1.50 GiB (384 extents).
  Logical volume vg_datos/lv_datos successfully resized.
  LV       VG       Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_datos vg_datos -wi-ao---- 1.50g
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_datos-lv_datos 1014M  141M  874M  14% /datos
```

**Qué observar:** `lvs` dice 1.50g, `df` sigue en 1014M. El "recipiente" creció, pero el sistema de archivos no sabe nada. Esto es lo que llega como ticket: "amplié el disco y sigue lleno".

3. Corregirlo: hacer crecer el XFS. XFS **solo** se hace crecer montado; por costumbre se le pasa el punto de montaje (en RHEL 9 `xfs_growfs` también acepta el dispositivo, siempre que esté montado):

```bash
sudo xfs_growfs /datos
df -h /datos
```

```text
meta-data=/dev/mapper/vg_datos-lv_datos isize=512    agcount=4, agsize=65536 blks
...
data blocks changed from 262144 to 393216
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_datos-lv_datos  1.5G  141M  1.4G  10% /datos
```

Para ext4 el equivalente es `sudo resize2fs /dev/vg_datos/lv_x` (funciona montado o desmontado).

4. Ahora bien hecho, en un solo paso con `-r`:

```bash
sudo lvextend -L +512M -r /dev/vg_datos/lv_datos
df -h /datos
sudo lvs vg_datos
sudo vgs vg_datos
```

```text
  Size of logical volume vg_datos/lv_datos changed from 1.50 GiB (384 extents) to 2.00 GiB (512 extents).
  Logical volume vg_datos/lv_datos successfully resized.
meta-data=/dev/mapper/vg_datos-lv_datos isize=512    agcount=6, agsize=65536 blks
...
data blocks changed from 393216 to 524288
Filesystem                     Size  Used Avail Use% Mounted on
/dev/mapper/vg_datos-lv_datos  2.0G  141M  1.9G   7% /datos
  LV       VG       Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_datos vg_datos -wi-ao---- 2.00g
  VG       #PV #LV #SN Attr   VSize VFree
  vg_datos   2   1   0 wz--n- 3.99g 1.99g
```

Los archivos siguen ahí y nadie tuvo que desmontar nada:

```bash
ls /datos/empresa/clientes/
```

Variantes que hay que conocer (**no ejecutar ahora**, necesitamos el espacio libre para el reto):
- `sudo lvextend -L 3G -r /dev/vg_datos/lv_datos` → tamaño absoluto (3 GiB) en vez de incremento.
- `sudo lvextend -l +100%FREE -r /dev/vg_datos/lv_datos` → usar todo lo que queda libre en el VG.
- `sudo lvextend -l +50%FREE -r ...` → la mitad de lo libre.

5. Inventario: qué dispositivos son PV y qué VG los usa. La vista más fiable es `lsblk -f` (muestra `LVM2_member`) junto con `pvs`:

```bash
lsblk -f -o NAME,FSTYPE,SIZE,MOUNTPOINTS /dev/sdb
sudo pvs
```

```text
NAME                  FSTYPE        SIZE MOUNTPOINTS
sdb                                   5G
├─sdb1                xfs          512M /archivos
├─sdb2                swap         512M [SWAP]
├─sdb3                LVM2_member    2G
│ └─vg_datos-lv_datos xfs            2G /datos
└─sdb4                LVM2_member    2G
  └─vg_datos-lv_datos xfs            2G /datos
  PV         VG       Fmt  Attr PSize   PFree
  /dev/sda2  rhel     lvm2 a--  <19.00g     0
  /dev/sdb3  vg_datos lvm2 a--   <2.00g     0
  /dev/sdb4  vg_datos lvm2 a--   <2.00g  1.99g
```

**Qué observar:** `lv_datos` cuelga de `sdb3` **y** de `sdb4`: sus extents están repartidos entre los dos PV. Eso es lo que LVM permite y una partición clásica no.

(El comando histórico `sudo lvmdiskscan` lista todos los dispositivos y marca cuáles son PV; en RHEL 9 está en desuso y, con el *devices file* activo, puede omitir los discos que LVM no tiene registrados. ⚠️ Verificar en la VM antes de la clase si sigue disponible en la versión de `lvm2` instalada; si no, no mencionarlo.)

- **Checkpoint:**

```bash
sudo lvs -o lv_name,lv_size vg_datos ; df -h /datos | tail -1
```

```text
  LV       LSize
  lv_datos 2.00g
/dev/mapper/vg_datos-lv_datos  2.0G  141M  1.9G   7% /datos
```

### Lab 3.3 — Snapshot, renombrar, borrar y liberar (12 min)

- **Objetivo:** ver un snapshot LVM funcionando (respaldo instantáneo antes de un cambio), practicar `lvrename`, `lvcreate -l 100%FREE` y `lvremove`, y conocer el orden de `vgremove`/`pvremove`.

1. Crear un snapshot de `lv_datos` (copia congelada en el tiempo; solo guarda los bloques que cambian, por eso puede ser pequeño):

```bash
sudo lvcreate -s -n lv_datos_snap -L 200M /dev/vg_datos/lv_datos
sudo lvs vg_datos
```

```text
  Logical volume "lv_datos_snap" created.
  LV            VG       Attr       LSize   Pool Origin   Data%  Meta%  Move Log Cpy%Sync Convert
  lv_datos      vg_datos owi-aos---   2.00g
  lv_datos_snap vg_datos swi-a-s--- 200.00m      lv_datos 0.01
```

**Qué observar:** `o` en `lv_datos` = tiene snapshots (*origin*); `s` en el snapshot; `Data%` es cuánto del espacio del snapshot se ha usado. Si llega a 100 % el snapshot se **invalida** (por eso en producción se vigila o se activa el autoextend en `lvm.conf`).

2. "Romper" algo en el original y comprobar que el snapshot conserva el estado anterior. XFS exige `-o nouuid` para montar el snapshot porque tiene el **mismo UUID** que el original:

```bash
sudo rm /datos/empresa/clientes/contrato_3.txt
echo "cambio posterior al snapshot" | sudo tee -a /datos/empresa/clientes/contrato_1.txt > /dev/null
sudo mkdir -p /mnt/snap
sudo mount -o nouuid /dev/vg_datos/lv_datos_snap /mnt/snap
ls /mnt/snap/empresa/clientes/
cat /mnt/snap/empresa/clientes/contrato_1.txt
```

```text
contrato_1.txt  contrato_2.txt  contrato_3.txt  contrato_4.txt  contrato_5.txt
Contrato 1 - PanamaTech
```

En el snapshot siguen los cinco contratos y `contrato_1.txt` no tiene la línea añadida. Para recuperar un archivo se copia desde `/mnt/snap`; para revertir **todo** el LV al momento del snapshot existe `lvconvert --merge` (con el origen desmontado; solo mencionarlo).

3. Desmontar y eliminar el snapshot (los snapshots son temporales: ralentizan la escritura mientras existen):

```bash
sudo umount /mnt/snap
sudo lvremove /dev/vg_datos/lv_datos_snap
```

```text
Do you really want to remove active logical volume vg_datos/lv_datos_snap? [y/n]: y
  Logical volume "lv_datos_snap" successfully removed.
```

4. `-l 100%FREE`, renombrar y borrar: crear un LV que ocupe todo lo libre, renombrarlo y eliminarlo sin preguntar:

```bash
sudo lvcreate -n lv_pruebas -l 100%FREE vg_datos
sudo vgs vg_datos
sudo lvrename vg_datos lv_pruebas lv_temporal
sudo lvs vg_datos
sudo lvremove -y /dev/vg_datos/lv_temporal
sudo vgs vg_datos
```

```text
  Logical volume "lv_pruebas" created.
  VG       #PV #LV #SN Attr   VSize VFree
  vg_datos   2   2   0 wz--n- 3.99g    0
  Renamed "lv_pruebas" to "lv_temporal" in volume group "vg_datos"
  LV          VG       Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_datos    vg_datos -wi-ao---- 2.00g
  lv_temporal vg_datos -wi-a----- 1.99g
  Logical volume "lv_temporal" successfully removed
  VG       #PV #LV #SN Attr   VSize VFree
  vg_datos   2   1   0 wz--n- 3.99g 1.99g
```

**Qué observar:** VFree pasa a 0 y vuelve a 1.99g. `lvremove` de un LV **montado** falla con "Logical volume vg_datos/lv_x contains a filesystem in use": siempre `umount` primero (y quitar la línea de fstab).

5. Solo para conocerlos (**no ejecutar**: destruiría el trabajo del día): el orden de desmontaje completo de un VG es

```text
sudo umount /datos
sudo lvremove /dev/vg_datos/lv_datos
sudo vgremove vg_datos
sudo pvremove /dev/sdb3 /dev/sdb4
sudo wipefs -a /dev/sdb3        # borra cualquier firma que quede
```

Y para renombrar un VG: `sudo vgrename vg_datos vg_produccion` (después hay que actualizar fstab, porque `/dev/mapper/vg_datos-lv_datos` cambia de nombre; con UUID no habría que tocar nada: esa es la ventaja de UUID incluso con LVM).

- **Checkpoint:**

```bash
sudo vgs -o vg_name,pv_count,lv_count,vg_size,vg_free vg_datos
```

```text
  VG       #PV #LV VSize VFree
  vg_datos   2   1 3.99g 1.99g
```

---

## Bloque 4 — Stratis (25 min)

### Conceptos (5 min)

**Qué es Stratis:** una capa de gestión de almacenamiento de Red Hat que combina, con una sola herramienta, lo que en LVM son varios pasos: un demonio (`stratisd`) administra *pools* construidos sobre device-mapper y XFS con **thin provisioning** (aprovisionamiento delgado), snapshots baratos, caché y cifrado opcional.

- **Pool:** conjunto de discos (`stratis pool create pool1 /dev/sdX`). Equivale más o menos a un VG.
- **Filesystem:** un XFS delgado dentro del pool (`stratis filesystem create pool1 fs1`). Por defecto se presenta con un tamaño virtual de **1 TiB**, aunque el pool tenga 3 GiB: el espacio real se consume a medida que se escribe. Eso es thin provisioning, y su riesgo es evidente: `df` miente, hay que vigilar `stratis pool list`.
- Los dispositivos aparecen en `/dev/stratis/<pool>/<fs>`.
- Stratis **exige dispositivos sin firma**: si el disco tuvo algo (LVM, un FS viejo), primero `sudo wipefs -a /dev/sdX`.
- Para montar por fstab hace falta `x-systemd.requires=stratisd.service` en las opciones: si el demonio no ha arrancado cuando systemd intenta montar, el dispositivo no existe y se cae al *emergency mode*.
- Stratis es opcional en el examen RHCSA para RHEL 9 (fue retirado de la lista de objetivos; ver Notas para el instructor). Se enseña porque está en la ficha del cliente y porque es la vía que Red Hat propone para quienes no quieren administrar LVM a mano.

### Lab 4.1 — Pool, filesystem, montaje persistente, snapshot y ampliación (20 min)

- **Objetivo:** instalar Stratis, crear `pool1` sobre `sdc1`, un filesystem `fs1` montado persistentemente en `/stratis`, tomar un snapshot y ampliar el pool con `sdc2`.

1. Instalar y arrancar el demonio:

```bash
sudo dnf install -y stratis-cli stratisd
sudo systemctl enable --now stratisd
systemctl is-active stratisd
```

```text
...
Complete!
Created symlink /etc/systemd/system/multi-user.target.wants/stratisd.service → /usr/lib/systemd/system/stratisd.service.
active
```

Si `dnf` dice que no hay repositorios, la VM no está registrada: `sudo subscription-manager register` (repaso del Día 5).

2. Verificar que `sdc1` está limpio y crear el pool:

```bash
lsblk -f /dev/sdc1
sudo stratis pool create pool1 /dev/sdc1
sudo stratis pool list
```

`lsblk -f` no debe mostrar FSTYPE. Si mostrara algo (porque alguien probó antes), `sudo wipefs -a /dev/sdc1`. Salida del pool:

```text
Name   Total / Used / Free            Properties                                UUID                                   Alerts
pool1  3 GiB / 37.63 MiB / 2.96 GiB   ~Ca,~Cr, Op                               7c2e9b1a-4d5f-4e6a-8b7c-9d0e1f2a3b4c
```

**Qué observar:** ya hay unas decenas de MiB "usados" sin un solo archivo: es la metadata del pool (la cifra exacta varía con la versión de `stratisd`). En cuanto se cree el filesystem, ese "Used" saltará a ~546 MiB, porque el XFS delgado de 1 TiB escribe sus propios metadatos. `~Ca` = sin caché, `~Cr` = sin cifrado, `Op` = sobreaprovisionamiento permitido.

3. Crear el filesystem y montarlo:

```bash
sudo stratis filesystem create pool1 fs1
sudo stratis filesystem list
sudo mkdir /stratis
sudo mount /dev/stratis/pool1/fs1 /stratis
df -h /stratis
```

```text
Pool   Filesystem   Total / Used / Free               Created            Device                    UUID
pool1  fs1          1 TiB / 546 MiB / 1023.47 GiB     Sep 03 2026 11:05  /dev/stratis/pool1/fs1    2f1d0c9b-...
Filesystem                                                 Size  Used Avail Use% Mounted on
/dev/mapper/stratis-1-7c2e...-thin-fs-2f1d...              1.0T  7.2G  1017G   1% /stratis
```

**Qué observar:** `df` dice **1 TiB** sobre un disco de 3 GiB. Eso es thin provisioning. Las cifras de "usado" no coinciden entre `stratis` y `df` y es normal: `stratis` reporta el espacio **físico** que el pool ha entregado (unos 546 MiB), mientras que `df` reporta los metadatos del XFS de 1 TiB (unos 7 GiB), que en su mayoría todavía no ocupan disco real. Preguntar a la clase: "¿qué pasa si alguien copia 4 GiB aquí?" (se llena el pool y las escrituras fallan aunque `df` diga que sobra). `lsblk /dev/sdc` mostrará una pila de siete u ocho capas de device-mapper: no hay que entenderlas, Stratis las administra.

4. Escribir datos y hacer el montaje persistente. El UUID de fstab es el UUID del filesystem que muestra `stratis filesystem list` (es el mismo del XFS):

```bash
echo "archivo en stratis" | sudo tee /stratis/hola.txt
sudo blkid /dev/stratis/pool1/fs1
echo "UUID=$(sudo blkid -s UUID -o value /dev/stratis/pool1/fs1)  /stratis  xfs  defaults,x-systemd.requires=stratisd.service  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo umount /stratis
sudo mount -a
findmnt /stratis
```

```text
/dev/stratis/pool1/fs1: UUID="2f1d0c9b-..." TYPE="xfs"
UUID=2f1d0c9b-...  /stratis  xfs  defaults,x-systemd.requires=stratisd.service  0 0
TARGET   SOURCE                                        FSTYPE OPTIONS
/stratis /dev/mapper/stratis-1-...-thin-fs-2f1d...     xfs    rw,relatime,seclabel,...
```

5. Snapshot (instantáneo, ocupa solo lo que cambie) y montaje del snapshot. Stratis asigna al snapshot un UUID nuevo, así que **no** hace falta `nouuid` (si `mount` se quejara de UUID duplicado, agregar `-o nouuid`):

```bash
sudo stratis filesystem snapshot pool1 fs1 snap1
sudo stratis filesystem list
sudo rm /stratis/hola.txt
sudo mkdir -p /mnt/snap1
sudo mount /dev/stratis/pool1/snap1 /mnt/snap1
cat /mnt/snap1/hola.txt
sudo umount /mnt/snap1
```

```text
Pool   Filesystem   Total / Used / Free      Created            Device                     UUID
pool1  fs1          1 TiB / 546 MiB / ...    Sep 03 2026 11:05  /dev/stratis/pool1/fs1     2f1d...
pool1  snap1        1 TiB / 546 MiB / ...    Sep 03 2026 11:09  /dev/stratis/pool1/snap1   8a3b...
archivo en stratis
```

En Stratis un snapshot es un filesystem más: se puede montar, escribir y, si se quiere, sustituir al original. Para borrarlo: `sudo stratis filesystem destroy pool1 snap1`.

6. Ampliar el pool en caliente añadiendo `sdc2`:

```bash
sudo stratis pool add-data pool1 /dev/sdc2
sudo stratis pool list
sudo stratis blockdev list pool1
```

```text
Name   Total / Used / Free         Properties     UUID       Alerts
pool1  5 GiB / 548 MiB / 4.46 GiB  ~Ca,~Cr, Op    7c2e...
Pool Name   Device Node   Physical Size   Tier   UUID
pool1       /dev/sdc1     3 GiB           Data   ...
pool1       /dev/sdc2     2 GiB           Data   ...
```

**Qué observar:** el pool creció a 5 GiB sin tocar `fs1` (que ya "medía" 1 TiB). Comparar con LVM: `vgextend` + `lvextend -r` + `xfs_growfs` en un solo comando.

- **Checkpoint:**

```bash
sudo stratis pool list ; findmnt -n -o TARGET,FSTYPE /stratis
```

```text
Name   Total / Used / Free         Properties     UUID       Alerts
pool1  5 GiB / 548 MiB / 4.46 GiB  ~Ca,~Cr, Op    7c2e...
/stratis xfs
```

---

## Bloque 5 — VDO: deduplicación y compresión (10 min, demo del instructor)

### Conceptos y demo (10 min)

**Qué es VDO (Virtual Data Optimizer):** una capa que **deduplica** (si dos bloques son idénticos, guarda uno solo), **comprime** y elimina bloques de ceros de forma transparente. Ideal para repositorios de máquinas virtuales, imágenes de contenedores y respaldos, donde hay mucho contenido repetido. Cuesta CPU y RAM (el índice de deduplicación vive en memoria).

**Cambio importante RHEL 8 → RHEL 9:** en RHEL 8 se administraba con la herramienta `vdo create ...` (Python). En **RHEL 9 VDO es un tipo de volumen LVM** (`--type vdo`): se crea con `lvcreate`, se consulta con `lvs` y `vdostats`. La herramienta `vdo` antigua queda solo para importar volúmenes viejos (`lvm_import_vdo`).

**Por qué es demo:** VDO necesita espacio para su índice (con la configuración por defecto, unos 2.5 GiB) más *slabs* de 2 GiB. En un disco de 5 GiB el `lvcreate` falla por espacio insuficiente. El instructor lo demuestra con un cuarto disco de **10 GB** (`/dev/vdd` en su VM UTM). Quien quiera reproducirlo en casa añade un disco de 10 GB.

**Qué paquete y qué módulo, según la versión de RHEL 9:** en las primeras versiones (9.0–9.4) el módulo del kernel venía en `kmod-kvdo` y se instalaba junto con `vdo` y `lvm2`. En las versiones más recientes de RHEL 9 el código pasó a estar **dentro del kernel** como `dm-vdo` y `kmod-kvdo` ya no se instala. Comprobar antes de la clase cuál aplica:

```bash
modinfo dm_vdo 2>/dev/null | head -3 || modinfo kvdo | head -3
dnf list --available kmod-kvdo
```

⚠️ **Verificar en la VM antes de la clase.** Si `modinfo dm_vdo` responde, basta con `sudo dnf install -y lvm2 vdo` (sin `kmod-kvdo`); si no, se instala también `kmod-kvdo`.

Comandos que ejecuta el instructor (los participantes solo miran y anotan):

```bash
sudo dnf install -y lvm2 vdo          # añadir kmod-kvdo solo si el kernel no trae dm-vdo
sudo vgcreate vg_vdo /dev/vdd
sudo lvcreate --type vdo -n lv_vdo -L 8G -V 16G vg_vdo
```

```text
    The VDO volume can address 4 GB in 2 data slabs, each 2 GB.
    It can grow to address at most 16 TB of physical storage in 8192 slabs.
    If a larger maximum size might be needed, use bigger slabs.
  Logical volume "lv_vdo" created.
```

`-L 8G` es el tamaño **físico** (lo que realmente ocupa en el VG); `-V 16G` es el tamaño **virtual** que verá el sistema de archivos (se apuesta a que la deduplicación/compresión logre 2:1). Si la apuesta sale mal y el físico se llena, las escrituras fallan: hay que vigilar `vdostats`.

```bash
sudo lvs -a vg_vdo
sudo mkfs.xfs -K /dev/vg_vdo/lv_vdo
sudo mkdir /vdo
sudo mount /dev/vg_vdo/lv_vdo /vdo
sudo vdostats --human-readable
```

```text
  LV              VG     Attr       LSize  Pool   Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_vdo          vg_vdo vwi-a-v--- 16.00g vpool0        0.00
  vpool0          vg_vdo dwi------- 8.00g                 50.00
  [vpool0_vdata]  vg_vdo Dwi-ao---- 8.00g
Device                    Size      Used Available Use% Space saving%
vg_vdo-vpool0-vpool       8.0G      4.0G      4.0G  50%           N/A
```

(`mkfs.xfs -K` evita que mkfs descarte bloques uno a uno, lo que en VDO es lento e inútil. Los porcentajes de `Data%` y `Use%` son orientativos: dependen del tamaño del índice.) Ahora la prueba de deduplicación: cinco copias del mismo archivo de 200 MiB:

```bash
sudo dd if=/dev/urandom of=/vdo/base.img bs=1M count=200 status=none
for i in 1 2 3 4; do sudo cp /vdo/base.img /vdo/copia_$i.img; done
sync
df -h /vdo
sudo vdostats --human-readable
```

```text
/dev/mapper/vg_vdo-lv_vdo   16G  1.1G   15G   7% /vdo
Device                    Size      Used Available Use% Space saving%
vg_vdo-vpool0-vpool       8.0G      4.2G      3.8G  52%           79%
```

**Qué observar:** `df` cree que hay 1 GiB de datos; VDO gastó ~200 MiB reales. "Space saving 79 %". Con datos ya comprimidos (video, zip) el ahorro sería casi cero: VDO no es magia, solo sirve donde hay redundancia.

Para desactivar compresión o deduplicación en un volumen: `sudo lvchange --compression n vg_vdo/lv_vdo`, `sudo lvchange --deduplication n vg_vdo/lv_vdo`. Para el fstab de un VDO se usa UUID y `defaults`, igual que cualquier LV.

---

## Bloque 6 — Automontaje (10 min, concepto)

### Conceptos (10 min)

**El problema:** un servidor con 30 recursos NFS montados permanentemente arranca lento (espera a cada servidor), se cuelga si uno de ellos está caído y mantiene conexiones que casi nunca se usan. **La solución** es montar **bajo demanda**: el directorio está vacío hasta que alguien entra en él; en ese momento se monta, y tras unos minutos sin uso se desmonta solo.

**autofs** (paquete `autofs`, servicio `autofs.service`): el automontador clásico de Linux. Tres tipos de mapas:

- **Mapa maestro** (`/etc/auto.master` o, mejor, un archivo propio en `/etc/auto.master.d/*.autofs`): dice qué directorio vigilar y en qué mapa están los detalles.
  ```text
  /misc     /etc/auto.misc          # mapa indirecto: todo lo que se pida bajo /misc
  /-        /etc/auto.direct        # mapa directo: rutas absolutas completas
  ```
- **Mapa indirecto** (`/etc/auto.misc`): las claves son subdirectorios del punto vigilado. Entrar a `/misc/datos` monta `servidor:/exports/datos`.
  ```text
  datos     -rw,sync    rhel01:/exports/datos
  ```
- **Mapa directo** (`/etc/auto.direct`): la clave es la ruta absoluta. Sirve cuando el punto de montaje ya tiene otras cosas alrededor (por ejemplo `/home/ana`, sin que autofs controle todo `/home`).
  ```text
  /datos/compartido   -rw,sync    rhel01:/exports/compartido
  ```
- Un caso muy típico: `/home/*` con **wildcard**: `*  -rw  servidor:/home/&` monta el home de cada usuario cuando inicia sesión.
- Comandos: `sudo dnf install autofs`, `sudo systemctl enable --now autofs`, `sudo systemctl reload autofs` tras cambiar mapas, `ls /misc/datos` para disparar el montaje, `findmnt -t autofs,nfs4`.

> **Referencia cruzada:** el **Día 9** (RH134 M4 Servicios de red: NFS) se instala `nfs-utils`, se exporta `/exports/datos` desde la propia VM y se configura autofs con mapa indirecto y directo contra `rhel01` (localhost). Hoy solo se entiende el concepto; el objetivo RHCSA "Configure autofs" se completa ese día.

**Alternativa nativa de systemd: `x-systemd.automount` en fstab.** No requiere paquetes. Una sola línea convierte un montaje en bajo demanda:

```text
UUID=c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90  /archivos  xfs  defaults,nofail,noauto,x-systemd.automount,x-systemd.idle-timeout=60  0 0
```

- `noauto`: no se monta al arrancar; `x-systemd.automount`: systemd genera una unidad `archivos.automount` que vigila el directorio; `x-systemd.idle-timeout=60`: se desmonta tras 60 s sin uso.
- Después: `sudo systemctl daemon-reload && sudo systemctl start archivos.automount`; `findmnt /archivos` mostrará `autofs` como tipo hasta que alguien entre; `ls /archivos` dispara el montaje real.
- Sirve igual para NFS (`servidor:/export /mnt/nfs nfs defaults,_netdev,x-systemd.automount 0 0`). autofs sigue siendo más flexible (wildcards, mapas centralizados en LDAP), pero para dos o tres montajes la opción de systemd basta.

**Si sobra tiempo (mini-lab de 5 min):** aplicar la línea de arriba a `/archivos` (reemplazando la existente), `daemon-reload`, `umount /archivos`, `systemctl start archivos.automount`, `findmnt /archivos` (tipo `autofs`), `ls /archivos` y otra vez `findmnt` (ahora `xfs`). Al terminar, dejar la línea original: `sudo systemctl stop archivos.automount`, restaurar la línea en fstab, `sudo systemctl daemon-reload && sudo mount -a`. Así no confunde el reto.

---

## Reto individual (20 min)

**Ticket #2026-0614 — Volumen para respaldos y swap adicional**

> "El equipo de respaldos necesita un volumen dedicado en el servidor `rhel01`:
> - Un volumen lógico de **1 GB** llamado **`lv_backups`** en el grupo **`vg_datos`**, con sistema de archivos **ext4**, montado de forma **persistente** en **`/backups`**. Si el disco fallara, el servidor debe seguir arrancando.
> - Reinicie el servidor y demuestre con `df -h` que `/backups` quedó montado.
> - Además, agregue **512 MB de swap adicional** usando un **archivo** `/swapfile`, también persistente. Demuéstrelo con `swapon --show`."

**Segunda parte** (el instructor la anuncia cuando alguien termina la primera):

> "El equipo pide **500 MB más** en `/backups`. Ya están copiando archivos, así que **no se puede desmontar**. Demuestre el cambio con `df -h` antes y después."

Entregable en el chat: salida de `df -h /backups`, `swapon --show` y `sudo lvs vg_datos`.

**Reto opcional** (para quien termine antes; el instructor prepara la falla con `sudo fallocate -l 1800M /datos/.cache_old.img`):

> "El monitoreo dice que `/datos` está al 95 %, pero el equipo asegura que no ha copiado casi nada. Encuentre la causa y libere el espacio."

### Solución (para el instructor)

Primera parte:

```bash
sudo lvcreate -n lv_backups -L 1G vg_datos
sudo mkfs.ext4 /dev/vg_datos/lv_backups
sudo e2label /dev/vg_datos/lv_backups BACKUPS            # opcional
sudo mkdir /backups
echo "/dev/mapper/vg_datos-lv_backups  /backups  ext4  defaults,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo findmnt --verify
df -h /backups
```

(También es válido `UUID=$(sudo blkid -s UUID -o value /dev/vg_datos/lv_backups)` en el primer campo. En pass puede ir `0` o `2`; ambos aceptables.)

Archivo swap:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress
sudo chmod 600 /swapfile
sudo restorecon -v /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile  swap  swap  defaults  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
swapon --show
```

```text
NAME       TYPE      SIZE USED PRIO
/dev/dm-1  partition   2G   0B   -2
/dev/sdb2  partition 512M   0B   -3
/swapfile  file      512M   0B   -4
```

(Si se omite `chmod 600`, `mkswap` avisa "insecure permissions 0644, fix with: chmod 0600 /swapfile". `restorecon` deja el archivo con el contexto SELinux `swapfile_t` que define la política para `/swapfile`: no siempre es imprescindible, pero es la práctica correcta y evita sorpresas si el sistema se reetiqueta.)

Reinicio y verificación. Avisar antes: **la sesión SSH se corta**; esperar unos 60 segundos y volver a conectar. Quien pasado ese tiempo no logre reconectar por SSH, debe abrir la ventana de la VM (VirtualBox/UTM) y mirar la consola: lo más probable es que esté en *emergency mode* por una línea mala en `/etc/fstab`.

```bash
sudo findmnt --verify        # última verificación antes de reiniciar
sudo reboot
# reconectar: ssh -p 2222 student@localhost
df -h /backups /datos /archivos /stratis
swapon --show
```

Todo debe volver montado. Si alguien cae en *emergency mode*, aplicar la caja de emergencia del Lab 2.3: casi siempre es un UUID mal copiado o un directorio inexistente.

Segunda parte (en caliente):

```bash
df -h /backups
sudo lvextend -L +500M -r /dev/vg_datos/lv_backups
df -h /backups
sudo lvs vg_datos
```

```text
  Size of logical volume vg_datos/lv_backups changed from 1.00 GiB (256 extents) to <1.49 GiB (381 extents).
  Logical volume vg_datos/lv_backups successfully resized.
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/vg_datos-lv_backups is mounted on /backups; on-line resizing required
The filesystem on /dev/mapper/vg_datos-lv_backups is now 390144 (4k) blocks long.
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/vg_datos-lv_backups  1.5G   24K  1.4G   1% /backups
```

Nota para la corrección: `+500M` son 125 extents, así que `lvs` muestra `<1.49g`; quien haya usado `-L 1.5G` verá `1.50g`. Ambas respuestas cumplen el ticket. Quien olvide `-r` verá `df` sin cambios: pedirle que lo detecte y corrija con `sudo resize2fs /dev/vg_datos/lv_backups`.

Reto opcional:

```bash
df -h /datos                       # 95 %
sudo du -sh /datos/*               # solo ~101M en empresa: no explica el uso
sudo ls -la /datos                 # aparece .cache_old.img de 1.8G
sudo du -ah /datos | sort -h | tail -5
sudo rm /datos/.cache_old.img
df -h /datos
```

Lección: `du -sh /datos/*` no incluye archivos ocultos; `du -sh /datos` (sin asterisco) o `ls -la` sí. Si el espacio no se libera tras borrar, un proceso mantiene el archivo abierto: `sudo lsof +L1` o `sudo lsof | grep deleted` y reiniciar ese proceso.

---

## Cierre (10 min)

**Estado final esperado de los discos** (pegarlo en el chat como último checkpoint del día):

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sdb /dev/sdc
```

```text
NAME                    SIZE TYPE FSTYPE      MOUNTPOINTS
sdb                       5G disk
├─sdb1                  512M part xfs         /archivos
├─sdb2                  512M part swap        [SWAP]
├─sdb3                    2G part LVM2_member
│ ├─vg_datos-lv_datos     2G lvm  xfs         /datos
│ └─vg_datos-lv_backups 1.5G lvm  ext4        /backups
└─sdb4                    2G part LVM2_member
  └─vg_datos-lv_datos     2G lvm  xfs         /datos
sdc                       5G disk
├─sdc1                    3G part stratis
│ └─stratis-1-private-...  (varias capas device-mapper)
│   └─stratis-1-...-thin-fs-...  1T  xfs      /stratis
└─sdc2                    2G part stratis
```

(`lv_datos` aparece bajo `sdb3` y `sdb4` porque sus extents están repartidos entre los dos PV; `lv_backups` puede aparecer bajo uno u otro.)

```text
/etc/fstab final (líneas añadidas hoy):
UUID=<sdb1>                       /archivos  xfs   defaults,nofail                                0 0
UUID=<sdb2>                       swap       swap  defaults                                       0 0
/dev/mapper/vg_datos-lv_datos     /datos     xfs   defaults                                       0 0
UUID=<stratis fs1>                /stratis   xfs   defaults,x-systemd.requires=stratisd.service   0 0
/dev/mapper/vg_datos-lv_backups   /backups   ext4  defaults,nofail                                0 0
/swapfile                         swap       swap  defaults                                       0 0
```

Tres ideas para llevarse: (1) **UUID en fstab + `daemon-reload` + `mount -a` + `findmnt --verify` antes de reiniciar**; (2) **XFS crece, no se reduce; `lvextend -r` hace las dos cosas**; (3) **thin provisioning (Stratis, VDO) hace que `df` mienta: vigilar el pool**.

Tomar el snapshot **`dia06-fin`** con la VM apagada (`sudo poweroff`).

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `lsblk`, `lsblk -f`, `lsblk -o NAME,SIZE,FSTYPE,UUID,MOUNTPOINTS` | Ver discos, particiones, LVM y montajes en árbol; `-f` añade FSTYPE, LABEL y UUID |
| `sudo blkid /dev/sdb1` · `sudo blkid -s UUID -o value /dev/sdb1` | Ver firma, UUID y label de un dispositivo; con `-o value` solo el valor (para scripts/fstab) |
| `sudo fdisk -l /dev/sdb` | Listar la tabla de particiones de un disco |
| `sudo parted /dev/sdb` → `print`, `mklabel gpt`, `unit MiB`, `mkpart primary xfs 1MiB 513MiB`, `mkpart primary linux-swap ...`, `set N lvm on`, `rm N`, `quit` | Particionar en modo interactivo (los cambios se aplican al instante) |
| `sudo fdisk /dev/sdc` → `g`, `n`, `t`, `d`, `p`, `w`, `q` | Particionar en modo interactivo (los cambios se escriben solo con `w`) |
| `sudo udevadm settle` · `sudo partprobe /dev/sdb` | Esperar a udev / forzar relectura de la tabla de particiones |
| `sudo mkfs.xfs [-f] [-L LABEL] /dev/sdb1` | Crear XFS (`-f` sobrescribe una firma existente) |
| `sudo mkfs.ext4 /dev/sdb1` · `sudo mkfs.vfat /dev/sdb1` | Crear ext4 / FAT32 (vfat requiere `dnf install dosfstools`) |
| `sudo xfs_admin -L LABEL /dev/sdb1` · `sudo e2label /dev/sdb1 LABEL` | Poner label a XFS (desmontado, máx. 12 car.) / a ext4 |
| `sudo mount /dev/sdb1 /archivos` · `sudo mount -o ro ...` · `sudo mount -o remount,rw /archivos` | Montar; montar solo lectura; cambiar opciones sin desmontar |
| `sudo umount /archivos` · `sudo fuser -vm /archivos` · `sudo lsof /archivos` | Desmontar; ver quién lo tiene ocupado ("target is busy") |
| `findmnt /archivos` · `sudo findmnt --verify` | Ver un montaje con sus opciones; verificar la sintaxis de `/etc/fstab` |
| `df -h` · `sudo du -sh /ruta` · `sudo du -ah /ruta \| sort -h \| tail` | Espacio libre por FS; espacio usado por directorio (incluye ocultos si no se usa `*`) |
| `sudo vi /etc/fstab` → `UUID=... /punto xfs defaults,nofail 0 0` | Montaje persistente. Campos: dispositivo, punto, tipo, opciones, dump, pass |
| `sudo systemctl daemon-reload && sudo mount -a` | Recargar unidades generadas desde fstab y montar todo lo pendiente (probar antes de reiniciar) |
| `sudo mkswap /dev/sdb2` · `sudo swapon /dev/sdb2` · `sudo swapoff ...` · `sudo swapon -a` | Formatear y activar/desactivar swap; activar todas las de fstab |
| `swapon --show` · `free -m` | Ver swaps activas con prioridad; ver memoria y swap |
| `sudo dd if=/dev/zero of=/swapfile bs=1M count=512` · `chmod 600` · `restorecon -v` · `mkswap` · `swapon` | Archivo swap (línea fstab: `/swapfile swap swap defaults 0 0`). Con `dd`, nunca con `fallocate` |
| `sudo pvcreate /dev/sdb3` · `sudo pvs` · `sudo pvdisplay` · `sudo pvremove` | Volúmenes físicos LVM |
| `sudo vgcreate vg_datos /dev/sdb3` · `sudo vgs` · `sudo vgdisplay` · `sudo vgextend vg_datos /dev/sdb4` · `sudo vgrename` · `sudo vgremove` | Grupos de volúmenes |
| `sudo lvcreate -n lv_datos -L 1G vg_datos` · `-l 100%FREE` · `sudo lvs` · `sudo lvdisplay` · `sudo lvrename VG viejo nuevo` · `sudo lvremove [-y]` | Volúmenes lógicos |
| `sudo lvextend -L +512M -r /dev/vg_datos/lv_datos` · `sudo lvextend -l +100%FREE -r ...` | Ampliar LV **y** su sistema de archivos en caliente (`-r` = `--resizefs`) |
| `sudo xfs_growfs /datos` · `sudo resize2fs /dev/vg/lv` | Hacer crecer XFS (por punto de montaje) / ext4 a mano si se olvidó `-r` |
| `sudo lvcreate -s -n snap -L 200M /dev/vg/lv` · `sudo mount -o nouuid /dev/vg/snap /mnt` | Snapshot LVM; montar un snapshot XFS (mismo UUID que el origen) |
| `lsblk -f` · `sudo pvs` · `sudo lvmdevices` | Ver qué dispositivos son PV (`LVM2_member`); *devices file* de RHEL 9 (`/etc/lvm/devices/system.devices`); `lvmdiskscan` está en desuso |
| `sudo dnf install stratis-cli stratisd` · `sudo systemctl enable --now stratisd` | Instalar y arrancar Stratis |
| `sudo stratis pool create pool1 /dev/sdc1` · `pool list` · `pool add-data pool1 /dev/sdc2` · `blockdev list` | Pools Stratis |
| `sudo stratis filesystem create pool1 fs1` · `filesystem list` · `filesystem snapshot pool1 fs1 snap1` · `filesystem destroy` | Filesystems Stratis (thin, XFS, 1 TiB virtual) en `/dev/stratis/pool/fs` |
| fstab Stratis: `UUID=... /stratis xfs defaults,x-systemd.requires=stratisd.service 0 0` | Montaje persistente de Stratis (espera al demonio) |
| `sudo wipefs -a /dev/sdX` | Borrar firmas (necesario antes de dar un disco usado a Stratis) |
| `sudo lvcreate --type vdo -n lv_vdo -L 8G -V 16G vg_vdo` · `sudo vdostats --human-readable` · `mkfs.xfs -K` | VDO en RHEL 9 (LVM); estadísticas de ahorro |
| `x-systemd.automount,x-systemd.idle-timeout=60,noauto` (fstab) · autofs: `/etc/auto.master.d/*.autofs` + mapa | Automontaje bajo demanda (práctica con NFS el Día 9) |

---

## Notas para el instructor

**Este es el día que más hay que practicar antes.** Según el plan, el fin de semana previo se repite el flujo completo **dos veces** en la VM UTM (con `vdb`, `vdc` de 5 GB y `vdd` de 10 GB), cronometrando cada lab. La segunda vez, hacerlo a partir del snapshot `dia05-fin` para asegurarse de que el punto de partida es el mismo que tendrán los participantes.

- **Preparar antes de la clase:**
  - VM del instructor con snapshot `dia05-fin`, discos `vdb` (5 GB), `vdc` (5 GB) y `vdd` (10 GB, para VDO).
  - Verificar que `sudo dnf install stratis-cli stratisd vdo` resuelve en la VM registrada (están en BaseOS/AppStream de RHEL 9) y comprobar si esa versión de RHEL 9 necesita además `kmod-kvdo` o ya trae `dm-vdo` en el kernel (`modinfo dm_vdo`). Anotar la versión de `stratis --version` y de `parted --version` para que las salidas coincidan con lo mostrado.
  - Ejecutar el flujo completo (Labs 2.1 → 4.1 y el reto) y guardar la salida real en un archivo de texto: si un participante pregunta "¿es normal que me salga X?", se compara.
  - Probar la caja de emergencia: poner a propósito una línea con UUID falso **sin `nofail`** en fstab, reiniciar, entrar como root en la consola de UTM, corregir y arrancar. Hay que haberlo hecho al menos una vez para guiar con calma a quien le pase.
  - Tener listo el comando del reto opcional: `sudo fallocate -l 1800M /datos/.cache_old.img` (lv_datos es de 2 GiB; deja ~100 MiB libres).
  - Preparar la demo de VDO en un segundo terminal ya con `vg_vdo` creado, por si el tiempo aprieta: así solo se muestran `lvs -a`, `vdostats` y la copia de archivos.
  - Recordar a los participantes al inicio: **todos los `sd` son `vd` en UTM** y `/boot/efi` extra en UTM.

- **Qué estudiar si es nuevo en RHEL:**
  1. **parted y fdisk con GPT:** `man parted`, `man fdisk`. Practicar `mkpart` con `unit MiB`, borrar y recrear, y el mensaje de alineación. Entender que en GPT "primary" es solo un nombre.
  2. **LVM de punta a punta:** `man lvm` (visión general), `man lvcreate`, `man lvextend` (sección `-r/--resizefs`), `man lvs` (columna Attr). Hacer dos veces el ciclo crear → montar → extender sin `-r` → `xfs_growfs` → extender con `-r` → snapshot → borrar. Leer `lvs -o help | head -60` para ver que hay decenas de columnas.
  3. **fstab y systemd:** `man fstab`, `man systemd.mount` (buscar `nofail`, `x-systemd.requires`, `x-systemd.automount`, `x-systemd.idle-timeout`), `man findmnt` (`--verify`). Probar qué pasa con y sin `nofail` con un UUID inexistente.
  4. **XFS:** `man xfs_growfs`, `man xfs_admin`, `man mkfs.xfs` (opción `-K` y `-L`). Confirmar que `xfs_growfs` solo funciona con el FS montado (acepta el punto de montaje y, en RHEL 9, también el dispositivo montado) y que XFS no se reduce.
  5. **Stratis:** `man stratis`, `stratis --help`, `stratis pool --help`. Practicar `pool create`, `filesystem create`, `snapshot`, `add-data`, `filesystem destroy`, `pool destroy`, y qué error da si el dispositivo tiene firma (`wipefs -a`). ⚠️ Verificar en la VM antes de la clase el comportamiento del UUID en snapshots (si `mount` del snapshot pide `nouuid`, ajustar el paso 5 del Lab 4.1) y los MiB "usados" que muestra `stratis pool list` recién creado.
  6. **VDO en LVM:** guía "Deduplicating and compressing logical volumes on RHEL" de la documentación de RHEL 9. ⚠️ Verificar en la VM antes de la clase el tamaño mínimo con el disco de 10 GB: si `-L 8G` falla por espacio, subir a `-L 9G` o reducir el índice con `--vdosettings 'vdo_slab_size_mb=512'` (no hace falta explicarlo en la clase). Los porcentajes de `lvs -a` y `vdostats` mostrados en el material son orientativos: anotar los reales.

- **Errores frecuentes de los participantes y cómo resolverlos:**

| Síntoma | Causa | Solución |
|---|---|---|
| `lsblk` no muestra `sdb`/`sdc` | Discos no añadidos, o añadidos con la VM encendida | Apagar, añadir en el controlador SATA (VBox) o VirtIO (UTM), arrancar |
| `umount: target is busy` | Una shell está dentro del directorio, o `less`/`tail -f` abierto ahí | `sudo fuser -vm /punto`; `cd ~`; cerrar el programa; `umount` |
| `mount -a` da `can't find UUID=...` | UUID mal copiado, o se volvió a formatear (`mkfs`) después de escribir fstab | `sudo blkid` y corregir la línea; usar el `echo ... $(blkid -s UUID -o value ...)` del lab |
| Tras editar fstab, `mount` muestra "(hint) your fstab has been modified..." | Falta `systemctl daemon-reload` | `sudo systemctl daemon-reload` |
| `mount: /datos: mount point does not exist` | No se creó el directorio | `sudo mkdir /datos` |
| La VM arranca en *emergency mode* | Línea de fstab mala sin `nofail` | Caja de emergencia del Lab 2.3: root en consola, `vi /etc/fstab`, `daemon-reload`, `mount -a`, `systemctl default` |
| `lvcreate -L 2G` falla: "insufficient free space: 512 extents needed, but only 511 available" | El VG de 2 GiB pierde 1 PE en metadata | Usar `-L 1G`, `-l 100%FREE` o `-l 511` |
| `lvextend` "correcto" pero `df -h` no cambia | Se olvidó `-r` | `sudo xfs_growfs /punto` (XFS) o `sudo resize2fs /dev/vg/lv` (ext4) |
| `xfs_growfs: /dev/vg_datos/lv_datos is not a mounted XFS filesystem` | El sistema de archivos no está montado (XFS solo crece montado) | Montarlo y repetir; por costumbre pasar el punto de montaje: `sudo xfs_growfs /datos` |
| `mkfs.xfs: ... appears to contain an existing filesystem` | Ya había una firma | Confirmar que es el dispositivo correcto y usar `-f` |
| `pvcreate` "Device /dev/sdb3 excluded by a filter" o "not found" | Firma previa, partición en uso, o *devices file* de RHEL 9 | `sudo wipefs -a /dev/sdb3`; `sudo lvmdevices --adddev /dev/sdb3`; revisar que no esté montada |
| `mount` del snapshot XFS: "Filesystem has duplicate UUID" | Snapshot LVM comparte UUID con el origen | `sudo mount -o nouuid ...` |
| `stratis pool create` falla con "device ... has an existing signature" o "is already in use" | La partición tuvo un FS/PV antes, o se apuntó a `sdc` entero cuando ya tiene particiones | `sudo wipefs -a /dev/sdc1`; usar `/dev/sdc1`, no `/dev/sdc` |
| `stratis` responde "Failed to connect to stratisd" / D-Bus | El demonio no está corriendo | `sudo systemctl enable --now stratisd` |
| `df -h /stratis` dice 1.0T | Es normal: thin provisioning | Explicar; vigilar con `stratis pool list` |
| Después del reinicio `/stratis` no montó | Falta `x-systemd.requires=stratisd.service` en fstab, o stratisd deshabilitado | Añadir la opción; `systemctl is-enabled stratisd` |
| `mkswap: /swapfile: insecure permissions 0644` | Falta `chmod 600` | `sudo chmod 600 /swapfile` y repetir `mkswap` |
| `swapon: /swapfile: skipping - it appears to have holes` | Archivo creado con `truncate` o `fallocate` (extents no escritos en XFS) | Borrarlo y recrearlo con `dd if=/dev/zero ...` |
| `parted`: "Warning: The resulting partition is not properly aligned" | Inicio/fin no múltiplo de 1MiB (por ejemplo usar `0` como inicio) | `Ignore` en el lab; en producción usar límites en MiB como en el material |
| Alguien hizo `mklabel gpt` sobre `sda` | Confundió el disco (parted no pregunta) | Restaurar snapshot `dia05-fin`. Insistir en `lsblk` antes de cada `parted` |

- **Diferencias VirtualBox (x86_64) vs UTM (aarch64):**
  - Nombres: `sda/sdb/sdc` vs `vda/vdb/vdc`; particiones `sdb1` vs `vdb1`. Todo lo de LVM, Stratis y fstab es idéntico porque no depende del nombre del disco (otra razón para usar UUID y nombres de mapper).
  - Modelo en `fdisk -l`/`parted print`: `VBOX HARDDISK` vs `Virtio Block Device`.
  - UTM arranca por UEFI: hay `vda1` de 600M en `/boot/efi` (vfat), así que `/boot` es `vda2` y el PV del VG `rhel` es `vda3`. En VirtualBox (BIOS por defecto en la mayoría de versiones) son `sda1` y `sda2`; si la VM se creó con "Habilitar EFI", también habrá `sda1` en `/boot/efi` y el resto se desplaza un número. Mencionarlo cuando se lee el `lsblk` inicial para que nadie piense que le falta o le sobra una partición.
  - Cómo añadir discos: VirtualBox → Almacenamiento → Controlador SATA → Agregar disco duro; UTM → Unidades → Nuevo → VirtIO. En UTM el disco nuevo puede aparecer con un nombre distinto al esperado si se agregó antes que otro; siempre confirmar con `lsblk -d -o NAME,SIZE`.
  - VDO está disponible tanto en x86_64 como en aarch64, así que la demo debería funcionar en la VM UTM del instructor. ⚠️ Verificar en la VM antes de la clase: `sudo dnf install -y lvm2 vdo`, después `sudo modprobe dm_vdo || sudo modprobe kvdo` y `lsmod | grep -E "vdo|kvdo"`. Si la versión de RHEL 9 aún usa `kmod-kvdo` y el kernel se actualizó sin actualizar ese paquete, `modprobe` falla: actualizar todo (`sudo dnf update`), reiniciar y volver a probar, o hacer la demo con capturas de pantalla preparadas.

- **Preguntas probables y respuesta corta:**
  - *¿Por qué XFS no se puede reducir?* Por diseño: sus estructuras (grupos de asignación) se reparten por todo el dispositivo y no hay código para moverlas. Red Hat lo eligió aun así por rendimiento y escalabilidad. Solución: respaldar, recrear más pequeño, restaurar. Por eso se empieza con LV pequeños y se crece.
  - *¿Puedo agrandar `/` (root) si se llena?* Sí: es un LV del VG `rhel`. Añadir disco, `vgextend rhel /dev/sdX`, `lvextend -l +100%FREE -r /dev/rhel/root`. Sin reiniciar.
  - *¿UUID o `/dev/mapper/vg-lv` en fstab?* Ambos son estables. UUID funciona para todo; los nombres de mapper son más legibles. Nunca `/dev/sdb1` a secas.
  - *¿Cuánta swap poner?* Depende de la aplicación. Regla orientativa: 2×RAM si RAM < 2 GB, = RAM entre 2 y 8 GB, 4–8 GB por encima. Para hibernar (portátiles) se necesita ≥ RAM. Muchas bases de datos prefieren poca swap.
  - *¿Qué diferencia hay entre LVM y Stratis? ¿Cuál uso?* LVM es el estándar, está en todas partes y lo pide el RHCSA. Stratis simplifica (un comando, thin provisioning, snapshots baratos) pero es más nuevo y menos flexible. En la PGN: LVM para servidores; Stratis si se quiere probar en un entorno específico.
  - *¿Qué riesgo tiene el thin provisioning?* Que `df` muestre espacio que físicamente no existe. Si el pool se llena, las escrituras fallan en todos los filesystems del pool. Hay que monitorear `stratis pool list` (o `lvs` en thin pools LVM).
  - *¿Snapshot LVM = respaldo?* No. Está en el mismo disco; si el disco muere, mueren los dos. Sirve para tener un punto de retorno antes de un cambio y para hacer respaldos consistentes (se respalda el snapshot mientras el sistema sigue escribiendo en el origen).
  - *¿MB o MiB?* `parted` y `lvcreate` entienden ambas: `500MB` = 500×10⁶ bytes; `500MiB`/`500M` (en LVM) = 500×2²⁰. La diferencia (~5 %) explica muchos "me faltan extents".
  - *¿Por qué GPT si el disco es de 5 GB?* Porque es el estándar actual, no tiene el límite de 4 primarias, tiene tabla de respaldo y es lo que pide el examen ("MBR y GPT"). MBR solo por compatibilidad.
  - *¿VDO comprime todo?* Solo lo que tiene redundancia. Video, zip y bases de datos cifradas no ahorran nada y sí gastan CPU.

- **Relación con el examen RHCSA (EX200, RHEL 9)** — objetivos que toca este día:
  - "Configure local storage": listar, crear y borrar particiones en discos MBR y GPT; crear y quitar volúmenes físicos; asignar PV a grupos de volúmenes; crear y borrar volúmenes lógicos; **configurar el montaje al arranque por UUID o label**; **añadir particiones, volúmenes lógicos y swap sin destruir datos**.
  - "Create and configure file systems": crear, montar, desmontar y usar **vfat, ext4 y xfs**; **extender volúmenes lógicos existentes**; montar y desmontar NFS y **configurar autofs** (esto último se completa el Día 9).
  - **Stratis y VDO:** estaban en los objetivos del RHCSA 8 ("Manage layered storage", "Configure disk compression") y **se retiraron de la lista para RHEL 9**. Decirlo con cautela: "según la lista publicada por Red Hat para EX200 en RHEL 9 ya no aparecen; verifiquen la lista vigente en redhat.com antes de presentar". Se enseñan igual porque están en la ficha del cliente y son útiles.
  - Consejo de examen: en el examen todo se valida tras un reinicio. La rutina `daemon-reload` → `mount -a` → `findmnt --verify` → `reboot` → `df -h` es la que evita perder los puntos de almacenamiento (y de paso los del resto del examen, porque una VM en *emergency mode* no puntúa nada).

---

## Tarea y preparación para el día siguiente

1. **Snapshot `dia06-fin`** con la VM apagada. Si algo quedó a medias (por ejemplo `/backups` no montó), restaurar `dia05-fin` y repetir los Labs 2.1, 3.1 y el reto siguiendo el material: es la mejor práctica posible.
2. **Leer** `man lvm` (solo la descripción general y la lista de comandos) y `man fstab` completo (es corto).
3. **Practicar 20 minutos** el ciclo crear/borrar un LV, sin mirar el material:
   ```bash
   sudo lvcreate -n lv_prueba -L 400M vg_datos
   sudo mkfs.xfs /dev/vg_datos/lv_prueba        # el tamaño mínimo soportado por Red Hat para XFS es 300 MiB
   sudo mkdir -p /mnt/prueba && sudo mount /dev/vg_datos/lv_prueba /mnt/prueba
   df -h /mnt/prueba
   sudo lvextend -L +100M -r /dev/vg_datos/lv_prueba && df -h /mnt/prueba
   sudo umount /mnt/prueba && sudo lvremove -y /dev/vg_datos/lv_prueba
   sudo vgs vg_datos
   ```
   Al terminar, `vgs` debe mostrar el mismo VFree que al inicio (~516 MiB si se hizo el reto). Repetir hasta hacerlo de memoria en menos de dos minutos.
4. **Dejar montados** `/datos` y `/backups`: en los próximos días el script `backup.sh` escribirá en `/backups`, y el Día 9 se exportará `/datos` por NFS.
5. **Autoevaluación** (responder mentalmente antes de la próxima clase): ¿qué hace `-r` en `lvextend`? ¿por qué `nofail`? ¿qué comando verifica fstab antes de reiniciar? ¿por qué el snapshot XFS necesita `nouuid`? ¿qué muestra `df` en un filesystem Stratis y por qué?
6. No se necesitan discos nuevos para el Día 7.
