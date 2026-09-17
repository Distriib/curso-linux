# Comandos — particionar, formatear, montar, `fstab` y swap

## `parted` — particionar (los cambios se aplican al instante)

```bash
sudo parted /dev/sdb
```
El prompt cambia a `(parted)`. Adentro:

| Orden | Qué hace |
|---|---|
| `print` | mostrar la tabla de particiones |
| `mklabel gpt` | crear la tabla GPT (borra lo que había, **sin preguntar**) |
| `unit MiB` | trabajar en MiB |
| `mkpart primary xfs 1MiB 513MiB` | partición 1, de 1 a 513 MiB, pensada para XFS |
| `mkpart primary linux-swap 513MiB 1025MiB` | partición 2, para swap |
| `mkpart primary 3073MiB 100%` | hasta el final del disco |
| `set 3 lvm on` | marcar la partición 3 como "Linux LVM" |
| `rm 3` | borrar la partición 3 |
| `quit` | salir |

```
mkpart  primary  xfs  1MiB  513MiB
          │       │     │      └ fin
          │       │     └ inicio (nunca 0: el primer MiB es de la tabla)
          │       └ para qué va a servir (solo una nota; no crea nada)
          └ en GPT es solo el nombre de la partición
```

## `fdisk` — la alternativa (escribe solo al final, con `w`)

```bash
sudo fdisk /dev/sdc
```

| Tecla | Qué hace |
|---|---|
| `g` | nueva tabla GPT |
| `n` | nueva partición: pregunta número, inicio y fin. `Enter` toma lo propuesto; `+3G` = 3 GiB |
| `p` | mostrar |
| `d` | borrar |
| `t` | cambiar el tipo (`swap`, `lvm`; `L` lista) |
| `q` | salir **sin** guardar |
| `w` | escribir y salir |

Después de particionar con cualquiera de los dos:
```bash
sudo udevadm settle
lsblk /dev/sdb
sudo fdisk -l /dev/sdb
```

## Formatear y etiquetar

| Comando | Qué hace |
|---|---|
| `sudo mkfs.xfs /dev/sdb1` | crear XFS |
| `sudo mkfs.xfs -f -L ARCHIVOS /dev/sdb1` | `-f` pisa lo que había; `-L` pone la etiqueta al crear |
| `sudo xfs_admin -L ARCHIVOS /dev/sdb1` | etiqueta a un XFS ya creado (desmontado, 12 letras máximo) |
| `sudo mkfs.ext4 /dev/sdb1` | crear ext4 (si había algo, pregunta) |
| `sudo e2label /dev/sdb1 ARCHIVOS` | etiqueta a un ext4 |
| `sudo mkfs.vfat -n DATOS /dev/sdb1` | crear vfat (necesita `sudo dnf install dosfstools`) |
| `sudo blkid /dev/sdb1` | ver tipo, UUID y etiqueta |
| `sudo blkid -s UUID -o value /dev/sdb1` | solo el UUID |
| `sudo wipefs -a /dev/sdb1` | borrar la firma (dejar la partición como nueva) |

`mkfs` tarda un segundo: escribe la estructura, no borra byte a byte. Pero **todo lo que había se pierde** y el UUID cambia.

## Montar y desmontar

| Comando | Qué hace |
|---|---|
| `sudo mkdir /archivos` | la carpeta tiene que existir |
| `sudo mount /dev/sdb1 /archivos` | montar |
| `sudo mount -o ro /dev/sdb1 /archivos` | montar solo lectura |
| `sudo mount -o remount,rw /archivos` | cambiar opciones sin desmontar |
| `sudo umount /archivos` | desmontar (se escribe `umount`, sin la n) |
| `findmnt /archivos` | ¿está montado? con qué opciones |
| `df -h /archivos` | espacio |
| `sudo fuser -vm /archivos` | quién lo tiene ocupado (`target is busy`) |

## `/etc/fstab` — el montaje permanente

```
UUID=c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90  /archivos  xfs  defaults,nofail  0  0
  │                                           │        │         │          │  └ 6: orden de fsck al arrancar: 0 para XFS y swap
  │                                           │        │         │          └ 5: dump, siempre 0
  │                                           │        │         └ 4: opciones, separadas por coma
  │                                           │        └ 3: tipo
  │                                           └ 2: dónde (la carpeta tiene que existir)
  └ 1: qué (UUID=, LABEL=, o /dev/mapper/vg-lv; nunca /dev/sdb1)
```

| Opción | Qué hace |
|---|---|
| `defaults` | lectura y escritura, se monta al arrancar |
| `nofail` | si el disco no está, el arranque **sigue**. Sin esto, la VM espera 90 segundos y cae en modo de emergencia |
| `ro` | solo lectura |
| `noauto` | no montar al arrancar |
| `x-systemd.requires=stratisd.service` | esperar a ese servicio antes de montar (Stratis) |

Después de editar `fstab`, **siempre**, en este orden:
```bash
sudo systemctl daemon-reload
sudo mount -a
sudo findmnt --verify
```
`daemon-reload`: systemd convierte cada línea en una unidad `.mount` y tiene que releer. `mount -a`: monta todo lo de `fstab` que falte; si hay un error, sale **ahora** y no en el próximo arranque. `findmnt --verify`: revisa sintaxis, UUID y carpetas.

### Si la VM arranca en modo de emergencia

Pantalla que dice `You are in emergency mode` y pide `Give root password for maintenance`. SSH no funciona. En la **ventana de la VM**:

1. Escribir la contraseña de **root**.
2. `mount -o remount,rw /`
3. `vi /etc/fstab` → poner `#` al inicio de la línea que falla → `:wq`
4. `systemctl daemon-reload`
5. `mount -a` (sin errores) → `systemctl default`

## Swap

| Comando | Qué hace |
|---|---|
| `swapon --show` | qué swap hay activa |
| `free -m` | memoria y swap, en MB |
| `sudo mkswap /dev/sdb2` | formatear como swap |
| `sudo swapon /dev/sdb2` | activar |
| `sudo swapoff /dev/sdb2` | desactivar |
| `sudo swapon -a` | activar todas las de `fstab` |

Línea de `fstab` para swap: `UUID=...  swap  swap  defaults  0 0`.

Swap en **archivo**, cuando no hay partición libre (se hace en el reto):
```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress
sudo chmod 600 /swapfile
sudo restorecon -v /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
Línea de `fstab`: `/swapfile  swap  swap  defaults  0 0`. Con `dd`, nunca con `fallocate`.

```
dd  if=/dev/zero  of=/swapfile  bs=1M  count=512
       │             │            │       └ cuántos bloques: 512 × 1M = 512 MiB
       │             │            └ tamaño de cada bloque
       │             └ a dónde escribe (output file)
       └ de dónde lee (input file): /dev/zero da ceros sin fin
```
