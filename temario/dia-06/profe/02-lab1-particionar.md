# Lab — Particionar `sdb` con `parted` y `sdc` con `fdisk` (comandos)

Vos en UTM tipeás `vdb` y `vdc`. Antes de abrir `parted`, `lsblk` en voz alta: *"confirmo que `sdb` es el vacío de 5G"*.

## Parte 1
```bash
lsblk
sudo parted /dev/sdb
```

## Parte 2
```
print
mklabel gpt
unit MiB
print
```
El `Error: unrecognised disk label` del primer `print` es lo esperado: no había tabla.

## Parte 3
```
mkpart primary xfs 1MiB 513MiB
mkpart primary linux-swap 513MiB 1025MiB
print
```
Ellos solos:
```
mkpart primary 1025MiB 2049MiB
print
```
Si `parted` avisa `not properly aligned for best performance`: responder `Ignore` (pasa si alguien escribió un número que no es múltiplo de 1MiB).

## Parte 4
```
rm 3
mkpart primary 1025MiB 3073MiB
```
Ellos solos:
```
mkpart primary 3073MiB 100%
set 3 lvm on
set 4 lvm on
print
quit
```
Si alguien tiene una partición con otros límites: `rm N` y volver a crearla; no hay que empezar de cero.

## Parte 5
```bash
sudo udevadm settle
lsblk /dev/sdb
sudo fdisk -l /dev/sdb
```
Si a alguien `lsblk` no le muestra `sdb1` a `sdb4`: `sudo partprobe /dev/sdb` y de nuevo `lsblk`.

## Parte 6
```bash
sudo fdisk /dev/sdc
```
Teclas, en orden: `g` · `n` `Enter` `Enter` `+3G` · `n` `Enter` `Enter` `Enter` · `p` · `w`.
Decir antes del `w`: *"hasta acá no se escribió nada; si se equivocaron, `q` y de nuevo"*.

## Parte 7
```bash
sudo udevadm settle
lsblk /dev/sdb /dev/sdc
```

Si alguien hizo `mklabel gpt` sobre `sda`: apagar, restaurar el snapshot `dia05-fin`, volver a agregar los dos discos y rehacer este lab. Mientras, sigue mirando.
