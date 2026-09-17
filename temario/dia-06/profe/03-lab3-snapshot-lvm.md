# Lab — Snapshot, renombrar y borrar (comandos)

## Parte 1
```bash
sudo lvcreate -s -n lv_datos_snap -L 200M /dev/vg_datos/lv_datos
sudo lvs vg_datos
```
Señalar la `o` de `lv_datos` (origen) y la `s` del snapshot.

## Parte 2
```bash
sudo rm /datos/empresa/clientes/contrato_3.txt
echo "cambio posterior al snapshot" | sudo tee -a /datos/empresa/clientes/contrato_1.txt
sudo mkdir -p /mnt/snap
sudo mount -o nouuid /dev/vg_datos/lv_datos_snap /mnt/snap
ls /mnt/snap/empresa/clientes/
cat /mnt/snap/empresa/clientes/contrato_1.txt
cat /datos/empresa/clientes/contrato_1.txt
```
Si alguien olvida `nouuid`: `mount: /mnt/snap: wrong fs type, bad option, bad superblock...`. Repetir con `-o nouuid`.

## Parte 3
```bash
sudo cp /mnt/snap/empresa/clientes/contrato_3.txt /datos/empresa/clientes/
ls /datos/empresa/clientes/
sudo umount /mnt/snap
sudo lvremove /dev/vg_datos/lv_datos_snap
```
Responder `y`.
```bash
sudo lvs vg_datos
```
Decir: *"los snapshots son temporales: mientras existen, cada escritura al original cuesta el doble. Se usan y se borran."*

## Parte 4
```bash
sudo lvcreate -n lv_pruebas -l 100%FREE vg_datos
sudo vgs vg_datos
```
Ellos solos:
```bash
sudo lvrename vg_datos lv_pruebas lv_temporal
sudo lvs vg_datos
sudo lvremove -y /dev/vg_datos/lv_temporal
sudo vgs vg_datos
```
Al final `VFree` tiene que volver a `1.99g`: ese espacio lo usa el reto. Si alguien apunta el `lvremove` a `lv_datos` en vez de `lv_temporal`, LVM se niega porque está montado (`contains a filesystem in use`): no se rompe nada.
