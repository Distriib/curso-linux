# Lab — Ampliar en caliente (comandos)

Vos en UTM: `vdb4`. Leer el ticket en voz alta antes de empezar. `/datos` queda montado todo el lab: nadie hace `umount`.

## Parte 1
```bash
sudo pvcreate /dev/sdb4
sudo vgextend vg_datos /dev/sdb4
sudo vgs vg_datos
sudo pvs
```

## Parte 2
```bash
sudo lvextend -L +512M /dev/vg_datos/lv_datos
sudo lvs vg_datos
df -h /datos
```
Decir: *"miren `lvs`: 1.50g. Miren `df`: 1014M. Esto es lo que llega como ticket."* Dejar que lo vean antes de seguir.

## Parte 3
```bash
sudo xfs_growfs /datos
df -h /datos
```
Si alguien pasa el dispositivo en vez del punto de montaje y le dice `is not a mounted XFS filesystem`: es que no está montado; con el punto de montaje no falla.

## Parte 4
```bash
sudo lvextend -L +512M -r /dev/vg_datos/lv_datos
df -h /datos
sudo lvs vg_datos
sudo vgs vg_datos
ls /datos/empresa/clientes/
```
Decir: *"`-r` hizo `xfs_growfs` solo. Una línea, las dos cosas."* No usar `-l +100%FREE` acá: el espacio libre es para el reto.

## Parte 5
```bash
lsblk -f /dev/sdb
sudo pvs
```
Señalar que `vg_datos-lv_datos` aparece dos veces, bajo `sdb3` y bajo `sdb4`: *"un volumen repartido en dos particiones. Eso no lo puede hacer una partición común."*
