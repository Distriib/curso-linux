# Lab 2.1 — Particionar `sdb` con `parted` y `sdc` con `fdisk`

Vamos a dividir los dos discos nuevos. Todos tipean lo que yo tipeo; cuando dice **Ahora ustedes**, lo hacen solos y mandan foto. En UTM: `vdb` y `vdc`.

| Disco | Herramienta | Partición | De | A | Tipo |
|---|---|---|---|---|---|
| `sdb` | `parted` | `sdb1` | 1 MiB | 513 MiB | xfs |
| | | `sdb2` | 513 MiB | 1025 MiB | linux-swap |
| | | `sdb3` | 1025 MiB | 3073 MiB | lvm |
| | | `sdb4` | 3073 MiB | 100% | lvm |
| `sdc` | `fdisk` | `sdc1` | inicio | +3G | Linux filesystem |
| | | `sdc2` | siguiente | final | Linux filesystem |

---

## Parte 1 — ¿Cuál es el disco vacío? Abrir `parted` sobre ese, no sobre otro

```bash
lsblk
sudo parted /dev/sdb
```

**Comprobar:**
```
GNU Parted 3.5
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)
```
El prompt ahora es `(parted)`. **Mirar dos veces que dice `/dev/sdb`**: `parted` no pregunta antes de escribir. Si dice `sda`, `quit` y volver a abrir.

---

## Parte 2 — ¿Qué tabla de particiones tiene el disco? Crear una GPT

```
(parted) print
(parted) mklabel gpt
(parted) unit MiB
(parted) print
```

**Comprobar:**
```
Error: /dev/sdb: unrecognised disk label
...
Partition Table: gpt
Disk Flags:

Number  Start  End  Size  File system  Name  Flags
```
El primer `print` da error: no había tabla. Después de `mklabel gpt` ya hay tabla, vacía. `mklabel` se aplicó **al instante**.

---

## Parte 3 — Las dos primeras particiones: una para XFS y otra para swap

```
(parted) mkpart primary xfs 1MiB 513MiB
(parted) mkpart primary linux-swap 513MiB 1025MiB
(parted) print
```

Ahora ustedes: la tercera partición, **a propósito muy chica**, de `1025MiB` a `2049MiB`, sin tipo. `print`. Foto.

**Comprobar:**
```
Number  Start    End      Size     File system     Name     Flags
 1      1.00MiB  513MiB   512MiB   xfs             primary
 2      513MiB   1025MiB  512MiB   linux-swap(v1)  primary  swap
 3      1025MiB  2049MiB  1024MiB                  primary
```
La columna `File system` es solo la intención: todavía no hay ningún sistema de archivos. La partición 3 quedó de 1 GiB y la necesitamos de 2.

---

## Parte 4 — ¿Cómo se corrige una partición mal hecha? Borrar y volver a crear

```
(parted) rm 3
(parted) mkpart primary 1025MiB 3073MiB
```

Ahora ustedes: la cuarta, de `3073MiB` hasta el final (`100%`); marcar la 3 y la 4 como `lvm`; `print`; `quit`. Foto.

**Comprobar:**
```
Number  Start    End      Size     File system     Name     Flags
 1      1.00MiB  513MiB   512MiB   xfs             primary
 2      513MiB   1025MiB  512MiB   linux-swap(v1)  primary  swap
 3      1025MiB  3073MiB  2048MiB                  primary  lvm
 4      3073MiB  5119MiB  2046MiB                  primary  lvm
```
La 4 puede terminar uno o dos MiB antes o después: es normal. Al salir dice `Information: You may need to update /etc/fstab.`

---

## Parte 5 — ¿El sistema ya ve las cuatro particiones?

```bash
sudo udevadm settle
lsblk /dev/sdb
sudo fdisk -l /dev/sdb
```

**Comprobar:**
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0    5G  0 disk
├─sdb1   8:17   0  512M  0 part
├─sdb2   8:18   0  512M  0 part
├─sdb3   8:19   0    2G  0 part
└─sdb4   8:20   0    2G  0 part
```
```
Disklabel type: gpt
Device       Start      End Sectors  Size Type
/dev/sdb1     2048  1050623 1048576  512M Linux filesystem
/dev/sdb2  1050624  2099199 1048576  512M Linux swap
/dev/sdb3  2099200  6293503 4194304    2G Linux LVM
/dev/sdb4  6293504 10483711 4190208    2G Linux LVM
```
Ahora sí dice `Disklabel type: gpt`. Si `lsblk` no muestra las particiones: `sudo partprobe /dev/sdb` y de nuevo `lsblk`.

---

## Parte 6 — El otro disco, con `fdisk`: ¿en qué se diferencia?

```bash
sudo fdisk /dev/sdc
```

Adentro, tecla por tecla (lo que va después de `:` lo tipeás vos):
```
Command (m for help): g
Command (m for help): n
Partition number (1-128, default 1): Enter
First sector (2048-10485726, default 2048): Enter
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-10485726, default 10485726): +3G
Command (m for help): n
Partition number (2-128, default 2): Enter
First sector (6293504-10485726, default 6293504): Enter
Last sector, +/-sectors or +/-size{K,M,G,T,P} (6293504-10485726, default 10485726): Enter
Command (m for help): p
Command (m for help): w
```

**Comprobar:**
```
Device       Start      End Sectors Size Type
/dev/sdc1     2048  6293503 6291456   3G Linux filesystem
/dev/sdc2  6293504 10485726 4192223   2G Linux filesystem
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```
Hasta el `w` no se escribió nada en el disco. Con `q` se salía sin cambiar nada; `parted` no tiene eso.

---

## Parte 7 — Los dos discos, listos

```bash
sudo udevadm settle
lsblk /dev/sdb /dev/sdc
```

**Comprobar:**
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0    5G  0 disk
├─sdb1   8:17   0  512M  0 part
├─sdb2   8:18   0  512M  0 part
├─sdb3   8:19   0    2G  0 part
└─sdb4   8:20   0    2G  0 part
sdc      8:32   0    5G  0 disk
├─sdc1   8:33   0    3G  0 part
└─sdc2   8:34   0    2G  0 part
```
Seis particiones, ninguna con sistema de archivos todavía. Foto.
