# Lab — Formatear, etiquetar y montar a mano (comandos)

Vos en UTM: `vdb1`.

## Parte 1
```bash
sudo mkfs.xfs /dev/sdb1
```

## Parte 2
```bash
sudo blkid /dev/sdb1
sudo xfs_admin -L ARCHIVOS /dev/sdb1
sudo blkid /dev/sdb1
```

## Parte 3
```bash
sudo mkdir /archivos
sudo mount /dev/sdb1 /archivos
findmnt /archivos
df -h /archivos
```
`seclabel` en las opciones es SELinux (Día 8); no detenerse.

## Parte 4
```bash
echo "hola desde sdb1" | sudo tee /archivos/nota.txt
ls -l /archivos
sudo umount /archivos
ls -l /archivos
sudo mount /dev/sdb1 /archivos
ls -l /archivos
```
Ellos: lo mismo con su nombre en la nota. Decir: *"si alguien escribe en `/archivos` mientras está desmontado, ese archivo queda tapado al montar de nuevo y llena el disco de `/` sin que nadie lo vea. Error clásico."*

## Parte 5
```bash
sudo umount /archivos
sudo mkfs.ext4 /dev/sdb1
```
Responder `y` a `Proceed anyway?`.
```bash
sudo e2label /dev/sdb1 ARCHIVOS
sudo blkid /dev/sdb1
```
Si a alguien `umount` le dice `target is busy`, está parado en `/archivos`: `cd ~` y repetir. Se ve a fondo en el próximo lab.

## Parte 6
```bash
sudo mkfs.xfs /dev/sdb1
sudo mkfs.xfs -f -L ARCHIVOS /dev/sdb1
sudo blkid /dev/sdb1
```
Queda **desmontado**. No montar: el Lab 2.3 lo monta con `mount -a`.
