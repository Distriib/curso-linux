# Lab — Crear un volumen LVM y montarlo (comandos)

Vos en UTM: `vdb3`.

## Parte 1
```bash
sudo pvcreate /dev/sdb3
sudo pvs
```
Si `pvcreate` dice `excluded by a filter` o `Device not found`: `sudo wipefs -a /dev/sdb3` y repetir. Si dice que está en uso: revisar que en el Lab 2.2 no le hayan hecho `mkfs` y `mount` a la partición equivocada.

## Parte 2
```bash
sudo vgcreate vg_datos /dev/sdb3
sudo vgs
sudo vgdisplay vg_datos
```
Señalar en `vgdisplay`: `PE Size 4.00 MiB`, `Total PE 511`. *"511 × 4 = 2044 MiB. Ese es el `<2.00g`."*

## Parte 3
```bash
sudo lvcreate -n lv_datos -L 1G vg_datos
sudo lvs
ls -l /dev/vg_datos/lv_datos /dev/mapper/vg_datos-lv_datos
```
Si alguien pide `-L 2G`: `insufficient free space: 512 extents needed, but only 511 available`. Que use `-L 1G`; es justamente la lección.

## Parte 4
```bash
sudo mkfs.xfs /dev/vg_datos/lv_datos
sudo mkdir /datos
sudo vim /etc/fstab
```
`G` → `o` → escribir → `Esc` → `:wq`:
```
/dev/mapper/vg_datos-lv_datos  /datos  xfs  defaults  0 0
```
```bash
sudo systemctl daemon-reload
sudo mount -a
df -h /datos
```
Decir: *"esta línea es igual para todos: no tiene UUID. Los nombres de LVM no cambian."* Si `mount -a` dice `mount point does not exist`: falta el `mkdir /datos`.

## Parte 5
```bash
sudo mkdir -p /datos/empresa/clientes /datos/empresa/logs
echo "Contrato 1 - PanamaTech" | sudo tee /datos/empresa/clientes/contrato_1.txt
```
Ellos solos:
```bash
echo "Contrato 2 - PanamaTech" | sudo tee /datos/empresa/clientes/contrato_2.txt
echo "Contrato 3 - PanamaTech" | sudo tee /datos/empresa/clientes/contrato_3.txt
ls /datos/empresa/clientes/
```
Va con `sudo tee` porque `/datos` es de root: `echo > archivo` con `sudo` delante falla (lo vieron el Día 2).

## Parte 6
```bash
sudo dd if=/dev/zero of=/datos/empresa/logs/app.log bs=1M count=100 status=none
sudo du -sh /datos/empresa
df -h /datos
```
Decir antes del `dd`: *"miren dos veces el `of=`: `dd` escribe donde le digan, sin preguntar."*

## Parte 7
```bash
lsblk -f /dev/sdb
```
