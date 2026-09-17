# 3 — LVM (guía del instructor)

Diez minutos, después del descanso. Lo único que se corre acá: `sudo pvs`, `sudo vgs`, `sudo lvs`, para que vean el VG `rhel` del instalador. Tres cosas que tienen que quedar: (1) tres capas, PV → VG → LV, y la letra + `s` para ver cada una; (2) ampliar es `vgextend` + `lvextend -r`, y sin `-r` el `df` no cambia; (3) `-L` es tamaño, `-l` es extents o porcentaje.

**Qué es LVM:** *Logical Volume Manager*. Una capa entre las particiones y los sistemas de archivos: junta discos en una bolsa y de ahí corta volúmenes que se pueden agrandar sin desmontar. Es lo que usa RHEL para `/` desde la instalación.
**Qué es un PV (volumen físico):** una partición o disco entero entregado a LVM con `pvcreate`. `pvcreate` escribe una cabecera al inicio; desde ahí `blkid` y `lsblk -f` lo muestran como `LVM2_member`.
**Qué es un VG (grupo de volúmenes):** la bolsa donde se juntan uno o más PV. Se ve como un solo espacio. Se crea con `vgcreate nombre PV` y crece con `vgextend nombre otroPV`.
**Qué es un LV (volumen lógico):** lo que se corta del VG. Se comporta como una partición: se formatea y se monta. `lvcreate -n nombre -L tamaño VG`.
**Qué es un extent (PE):** la unidad mínima en que el VG reparte el espacio: 4 MiB. Todo tamaño se redondea a extents. El VG se queda 1 MiB para su propia información, y por eso una partición de 2048 MiB da 511 extents (2044 MiB), no 512.
**Qué es el `<` en `<2.00g`:** "un poco menos de". LVM lo pone cuando el tamaño real no es redondo. `<2.00g` = 2044 MiB. Es la explicación de por qué `-L 2G` falla ahí.
**Qué es `-n`:** *name*: el nombre del LV.
**Qué es `-L`:** tamaño en unidades: `1G`, `512M`. Con `+` delante (`-L +512M`) es "agregar eso a lo que tiene", en `lvextend`.
**Qué es `-l` (ele minúscula):** tamaño en extents o en porcentaje: `-l 256` (extents), `-l 100%FREE` (todo lo libre del VG), `-l 50%VG` (la mitad del VG). Sirve cuando el tamaño exacto no cuadra en extents.
**Qué es `pvs` / `vgs` / `lvs`:** resumen de una línea por objeto. `pvdisplay` / `vgdisplay` / `lvdisplay`: el detalle completo. Todos con `sudo`.
**Qué es `/dev/mapper/vg_datos-lv_datos` y `/dev/vg_datos/lv_datos`:** los dos nombres del mismo LV. Los dos son enlaces simbólicos (Día 2) al dispositivo real `/dev/dm-2`. El de `mapper` es el que se pone en `fstab`; el otro es más cómodo para tipear en los comandos.
**Qué es `dm-2`:** el tercer dispositivo del device mapper (`dm-0` es `rhel-root`, `dm-1` es `rhel-swap`). Es el nombre interno; no se usa en `fstab` porque el número puede cambiar.
**Qué es `vgextend`:** agregar un PV a un VG que ya existe. Si la partición todavía no es PV, la convierte solo; en el lab se hace `pvcreate` aparte para que se vea el paso.
**Qué es `lvextend`:** agrandar un LV. Con `-L +512M` agrega; con `-L 3G` lo lleva a ese tamaño; con `-l +100%FREE` usa todo lo libre.
**Qué es `-r` (`--resizefs`):** que `lvextend` agrande también el sistema de archivos que está encima, llamando a `xfs_growfs` o `resize2fs` según corresponda. Sin `-r`, el LV crece pero el sistema de archivos sigue del tamaño viejo y `df -h` no cambia.
**Qué es `xfs_growfs`:** agrandar un XFS hasta ocupar todo su dispositivo. Solo funciona **montado**; se le pasa el punto de montaje (`xfs_growfs /datos`).
**Qué es `resize2fs`:** lo mismo para ext4; se le pasa el dispositivo (`resize2fs /dev/vg_datos/lv_backups`). Funciona montado o desmontado.
**Qué es "en caliente":** con el volumen montado y en uso, sin cortar el servicio. Todo lo de hoy en LVM es en caliente, salvo achicar (que no se practica).
**Qué es la columna `Attr` de `lvs`:** diez letras que describen el LV. Las que importan: primera letra `-` normal, `o` origen de snapshot, `s` snapshot; `w` escritura; `a` activo; `o` en la sexta posición = abierto (montado). `-wi-ao----` = normal, escribible, activo, montado.
**Qué es un snapshot LVM:** una copia congelada de un LV en un instante. No copia todo: guarda solo los bloques que cambian en el original después de la foto (por eso puede medir 200M para un LV de 2G). Si esos cambios superan su tamaño, el snapshot se invalida. Se crea con `lvcreate -s`, se monta como cualquier LV y se borra con `lvremove`. No es respaldo: está en el mismo disco.
**Qué es `Data%` en `lvs`:** en un snapshot, cuánto de su espacio ya se gastó. Si llega a 100, se invalida.
**Qué es `nouuid`:** opción de montaje de XFS para montar un sistema de archivos cuyo UUID ya está montado en otro lado. El snapshot tiene el mismo UUID que el original (es una copia exacta); sin `nouuid`, `mount` falla y el registro dice `Filesystem has duplicate UUID`.
**Qué es `lvrename`:** renombrar un LV: `lvrename VG nombre_viejo nombre_nuevo`. Si estaba en `fstab`, hay que corregir la línea.
**Qué es `lvremove`:** borrar un LV. Pregunta `Do you really want to remove...? [y/n]`; con `-y` no pregunta. Falla si está montado: primero `umount` (y sacar la línea de `fstab`).
**Qué es `vgrename` / `vgremove` / `pvremove`:** renombrar un VG (y corregir `fstab`), borrar un VG vacío, quitar la cabecera LVM de un PV. El orden de destrucción es el inverso al de creación: `umount`, `lvremove`, `vgremove`, `pvremove`.

---

## Tres capas

Leer el diagrama de izquierda a derecha con la analogía de los ladrillos: *"bolsas (PV), bodega (VG), paredes (LV)"*. Después la tabla, señalando una fila completa: *"crear, ver, ampliar, borrar. Las tres capas tienen los mismos verbos."*

`sudo pvs`, `sudo vgs`, `sudo lvs`.
**Qué señalar:** `/dev/sda2` es PV del VG `rhel`; `rhel` tiene dos LV, `root` y `swap`. *"Esto lo hizo el instalador el Día 1. Hoy hacemos lo mismo a mano con `sdb3`."*

---

## Anatomía

**Qué decir:** *"`-L` mayúscula es tamaño: 1G, 512M. `-l` minúscula es extents o porcentaje: `100%FREE` es 'todo lo que queda'."*
**La frase que hay que decir:** *"el VG reparte en pedazos de 4 MiB y se queda 1 MiB para él. Por eso una partición de 2 GiB da un VG de 'un poco menos de 2', el `<2.00g`. Si piden `-L 2G` ahí, falla por unos pocos extents. Se pide `-L 1G`, o `-l 100%FREE` para 'todo'."*

---

## Dónde aparece un LV

**Qué señalar:** los dos nombres, mismo `dm-2`. *"En `fstab` va el de `mapper`. Y no hace falta UUID: los nombres de LVM no cambian entre arranques."*

---

## Ampliar en caliente

**Esto es lo más importante del bloque. Decirlo despacio:** *"ampliar son dos pasos: agrandar el recipiente (`lvextend`) y agrandar el sistema de archivos adentro. La `-r` hace los dos. Sin `-r`, el LV dice 1.5G y `df` sigue diciendo 1G. Es el ticket más común: 'amplié el disco y sigue lleno'."* En el Lab 3.2 lo hacen mal a propósito primero, para verlo.
**Qué señalar:** la tabla de corrección: `xfs_growfs` con el punto de montaje; `resize2fs` con el dispositivo.

---

## Leer `lvs`

Señalar en el diagrama solo tres letras: `a` (activo), `o` en la sexta (montado) y la primera (`o` origen / `s` snapshot). No leer las demás.

---

## Snapshot

**Qué decir:** *"una foto del volumen en este instante. Se guarda solo lo que cambia después, por eso 200M alcanzan para un volumen de 2G mientras no cambie demasiado. Sirve para tener a dónde volver antes de un cambio, o para respaldar algo consistente mientras el sistema sigue escribiendo. No es un respaldo: si se muere el disco, se mueren los dos."*
**Qué señalar:** `nouuid`: *"el snapshot es copia exacta, UUID incluido; XFS no deja montar dos iguales sin esa opción."*

---

## Renombrar, borrar y el orden

**Qué decir:** *"se desarma al revés de como se armó: desmontar, borrar el LV, borrar el VG, quitar el PV. Al revés no funciona."* No se practica el desarme completo: destruiría el trabajo del día.
