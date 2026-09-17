# 1 — Cómo se organiza un disco (guía del instructor)

Quince minutos, tipeando mientras explicás. Los comandos están en la pestaña de estudiantes; acá solo se **lee** el disco, no se toca nada. Tres cosas que tienen que quedar: (1) la cadena disco → partición → sistema de archivos → punto de montaje, un comando por paso; (2) en `fstab` va el UUID, no `sdb`; (3) XFS crece pero no se achica, y LVM es lo que permite crecer sin apagar.

Antes de arrancar, decir: *"yo estoy en UTM y mis discos se llaman `vdb` y `vdc`. En sus pantallas, con VirtualBox, son `sdb` y `sdc`. El material dice `sdb`; yo voy a tipear `vdb`. Es la única diferencia."*

**Qué es un disco:** el dispositivo que guarda los datos, físico o virtual. Para el sistema es una tira de bytes sin ninguna estructura hasta que alguien se la pone.
**Qué es un dispositivo de bloques:** cómo el kernel presenta un disco: un archivo especial en `/dev` del que se lee y escribe por bloques (pedazos de 512 bytes o 4 KiB). `/dev/sdb` es un archivo, no una carpeta.
**Qué es el kernel:** el núcleo del sistema: el programa que habla con el hardware. Cuando aparece un disco, es el kernel el que lo ve y crea `/dev/sdb`.
**Qué es `/dev`:** la carpeta donde el kernel muestra el hardware como archivos. `sd` = disco SCSI/SATA (VirtualBox); `vd` = disco virtio (UTM; más rápido porque no simula hardware real). La letra es el orden: `a` el primero, `b` el segundo.
**Qué es `sr0`:** el lector de CD/DVD virtual. Se ignora.
**Qué es una partición:** un pedazo del disco con inicio y fin. Se numera: `sdb1`, `sdb2`. Sirve para separar usos (arranque, sistema, datos, swap) dentro de un mismo disco.
**Qué es la tabla de particiones:** el índice, al inicio del disco, que dice dónde empieza y termina cada partición. Sin tabla, el disco está "en blanco". MBR y GPT son dos formatos de ese índice.
**Qué es MBR:** el formato viejo (1983): 4 particiones máximo, discos hasta 2 TiB. `parted` lo llama `msdos`.
**Qué es GPT:** el formato actual: 128 particiones, discos de cualquier tamaño, con una copia de la tabla al final del disco. Hoy siempre GPT.
**Qué es un sistema de archivos (filesystem, FS):** la estructura que organiza archivos y carpetas dentro de una partición: dónde está cada archivo, su nombre, sus permisos. "Formatear" = crear un sistema de archivos. Sin FS, una partición es un pedazo de bytes que no sirve para guardar archivos.
**Qué es XFS:** el sistema de archivos de RHEL desde la versión 7. Rápido, escala a discos enormes, crece en caliente, **no se achica**.
**Qué es ext4:** el clásico de Linux. Crece y se achica (desmontado). En RHEL se usa por compatibilidad o cuando hace falta achicar.
**Qué es vfat:** el FAT32 de Windows y los pendrives. No guarda permisos ni dueño. Se usa para intercambiar con Windows y para la partición de arranque UEFI (`/boot/efi`).
**Qué es montar / punto de montaje:** montar es enganchar un sistema de archivos en una carpeta del árbol. La carpeta se llama punto de montaje. Desde ese momento, lo que hay adentro de la carpeta es lo que hay en ese disco. Desmontar = desenganchar: la carpeta queda vacía, los datos siguen en el disco.
**Qué es `lsblk`:** *list block devices*. Muestra discos, particiones y volúmenes en forma de árbol, con tamaño y punto de montaje. Es el primer comando del día y el que se corre antes de tocar cualquier disco.
**Qué es `lsblk -f`:** lo mismo, con el tipo de sistema de archivos (`FSTYPE`), la etiqueta y el UUID de cada uno.
**Qué es una firma:** los primeros bytes de una partición, que dicen qué hay adentro (XFS, ext4, swap, LVM). `blkid` y `lsblk -f` leen esa firma. Una partición nueva no tiene firma.
**Qué es `blkid`:** *block id*. Lee la firma de cada dispositivo y muestra tipo, UUID y etiqueta. Con `sudo` para ver todos.
**Qué es un UUID:** *Universally Unique Identifier*: un número de 32 dígitos hexadecimales en cinco grupos (`3f0c2b7e-9a1d-4c55-8a2f-1b7e6d0c9f41`) que recibe cada sistema de archivos al crearse. Es único y no cambia si el disco cambia de nombre. Sí cambia si se vuelve a formatear.
**Qué es una etiqueta (label):** un nombre corto que le ponés vos al sistema de archivos (`ARCHIVOS`). Más legible que el UUID; también sirve en `fstab`.
**Qué es `PARTUUID` / `PARTLABEL`:** el identificador y el nombre **de la partición** (los pone `parted`), distintos del UUID y la etiqueta del sistema de archivos. En `fstab` se usa el UUID del sistema de archivos.
**Qué es `df -h`:** *disk free*, en unidades legibles. Muestra el espacio de cada sistema de archivos **montado**. Lo que no está montado no aparece.
**Qué es `/boot`:** la partición con el kernel y lo necesario para arrancar. 1 GiB, XFS, la hizo el instalador.
**Qué es `/boot/efi`:** una partición vfat extra que necesitan las máquinas con arranque UEFI (UTM, y VirtualBox si se marcó "Habilitar EFI"). Si la ven, no es un error.
**Qué es `/dev/mapper` y `dm`:** *device mapper*: la capa del kernel que arma dispositivos virtuales a partir de otros. LVM la usa. `/dev/mapper/rhel-root` = grupo `rhel`, volumen `root`. `dm-0`, `dm-1` son sus nombres internos.
**Qué es LVM:** *Logical Volume Manager*: una capa entre las particiones y los sistemas de archivos que permite juntar varios discos y agrandar volúmenes sin apagar. El Bloque 3 es todo esto.
**Qué es PV / VG / LV:** volumen físico (una partición entregada a LVM), grupo de volúmenes (la bolsa donde se juntan los PV), volumen lógico (lo que se corta de la bolsa; encima va el sistema de archivos).
**Qué es `LVM2_member`:** lo que `lsblk -f` muestra como `FSTYPE` de una partición que es PV: no tiene sistema de archivos, pertenece a LVM.
**Qué es swap:** espacio en disco que el kernel usa como extensión de la RAM cuando se acaba. `rhel-swap` son los 2 GiB que puso el instalador. Se ve en el Bloque 2.
**Qué es MiB / GiB:** 1 MiB = 1024 × 1024 bytes; 1 MB = 1.000.000. Las herramientas de disco trabajan en MiB y GiB (`M`, `G`). La diferencia (5 %) es lo que explica "pedí 2G y no entra".
**Qué es un sector:** la unidad mínima del disco, 512 bytes. `fdisk` mide en sectores; por eso los números grandes.

---

## La cadena

**Qué decir:** *"un disco recién conectado es un terreno baldío. Para usarlo hay cuatro pasos, cada uno con su comando: lo dividís en lotes (partición), construís el edificio (sistema de archivos) y le ponés la dirección en la puerta (punto de montaje)."*
**Qué señalar:** la fila de la tabla que dice `mount`, `/etc/fstab`: *"montar a mano dura hasta el próximo reinicio; `fstab` es lo que lo hace permanente. Es la mitad del día."*

La frase de las letras de unidad: *"acá no hay `D:`. Un disco nuevo aparece en la carpeta que ustedes elijan. Si lo desmontan, la carpeta sigue existiendo, vacía."*

---

## Los nombres en `/dev`

`lsblk`.
**Qué señalar:** `sda` con `sda1` y `sda2` debajo; `sdb` y `sdc` solos. `TYPE` dice `disk`, `part`, `lvm`. La columna `MOUNTPOINTS`: `/boot`, `/`, `[SWAP]`; `sdb` y `sdc` no tienen nada.
**Qué decir:** *"`sdb` puede llamarse `sdc` mañana si alguien agrega un disco. Por eso en `fstab` no se usa el nombre, se usa el UUID. Lo vemos en el Bloque 2."*

---

## Lo que ya hizo el instalador

`lsblk -f`.
**Qué señalar:** `sda1` es `xfs` en `/boot`. `sda2` dice `LVM2_member`: **no tiene sistema de archivos**, está entregada a LVM. Debajo, `rhel-root` (xfs, `/`) y `rhel-swap`. *"El sistema ya usa LVM. Lo que aprendemos hoy sirve para agrandar `/` el día que se llene: se agrega un disco y se amplía, sin reinstalar."*
En UTM: `vda1` es `/boot/efi` (vfat), `vda2` es `/boot`, `vda3` el LVM. Decirlo para que nadie crea que le sobra una partición.

---

## GPT

No leer la tabla. **Qué decir:** *"dos formatos de índice. MBR es de 1983 y admite 4 particiones. GPT es el actual. Hoy siempre GPT; MBR solo si un equipo viejo lo exige."*

---

## Sistemas de archivos

**Qué señalar:** la fila "Achicar": `nunca` en XFS. **Decir tres veces en el día: XFS no se achica.** *"Si les quedó grande: respaldar, destruir, crear más chico, restaurar. Por eso se empieza chico y se crece, y crecer es fácil: lo van a ver en LVM."*
ext4 y vfat, una frase cada uno: *"ext4 cuando haga falta achicar o compatibilidad; vfat para pendrives, Windows y `/boot/efi`; para todo lo demás, XFS."*

---

## UUID y etiqueta

`sudo blkid`.
**Qué señalar:** cada línea tiene `UUID="..."` y `TYPE="..."`. En `sda1` aparece además `PARTUUID`: *"ese es de la partición, no del sistema de archivos; el que va en `fstab` es `UUID`."*
**La frase:** *"el UUID es la cédula del sistema de archivos. El nombre `sdb1` es el apodo. En `fstab` se usa la cédula."* Y: *"si vuelven a formatear, cambia la cédula. Después de cada `mkfs`, `blkid` otra vez."*

---

## Por qué LVM

Leer el diagrama de izquierda a derecha con la analogía: *"los PV son bolsas de ladrillos, el VG es la bodega donde se vacían todas las bolsas, los LV son las paredes que se construyen con esos ladrillos. Se pueden traer más bolsas a la bodega cuando quieran y alargar una pared sin tumbarla. Eso es `vgextend` y `lvextend`, Bloque 3."*
**Qué señalar:** que `sda2` ya es un PV del VG `rhel`: el sistema entero corre sobre LVM.

---

## El mapa de hoy

Leer la tabla completa: es el plan del día. *"`sdb` lo partimos en cuatro con `parted`: archivos, swap y dos pedazos para LVM. `sdc` lo partimos en dos con `fdisk` y va entero a Stratis. Cada fila es un lab."* Repetir: en UTM, `vdb` y `vdc`.
