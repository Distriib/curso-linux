# Lab 3.3 — Snapshot, renombrar y borrar

Vamos a sacarle una foto a `lv_datos` antes de romper algo, recuperar un archivo desde la foto, y practicar renombrar y borrar volúmenes. Todos tipean cada comando; cuando dice **Ahora ustedes**, lo hacen solos. Foto de cada parte.

---

## Parte 1 — ¿Cómo congelo el estado de un volumen?

```bash
sudo lvcreate -s -n lv_datos_snap -L 200M /dev/vg_datos/lv_datos
sudo lvs vg_datos
```

**Comprobar:**
```
  Logical volume "lv_datos_snap" created.
  LV            VG       Attr       LSize   Pool Origin   Data%
  lv_datos      vg_datos owi-aos---   2.00g
  lv_datos_snap vg_datos swi-a-s--- 200.00m      lv_datos 0.01
```
`o` en `lv_datos`: es origen de un snapshot. `s` en el nuevo: es un snapshot. `Data% 0.01`: casi nada cambió todavía.

---

## Parte 2 — Rompo el original: ¿el snapshot lo conserva?

```bash
sudo rm /datos/empresa/clientes/contrato_3.txt
echo "cambio posterior al snapshot" | sudo tee -a /datos/empresa/clientes/contrato_1.txt
sudo mkdir -p /mnt/snap
sudo mount -o nouuid /dev/vg_datos/lv_datos_snap /mnt/snap
ls /mnt/snap/empresa/clientes/
cat /mnt/snap/empresa/clientes/contrato_1.txt
cat /datos/empresa/clientes/contrato_1.txt
```

**Comprobar:**
```
contrato_1.txt  contrato_2.txt  contrato_3.txt
Contrato 1 - PanamaTech
Contrato 1 - PanamaTech
cambio posterior al snapshot
```
En `/mnt/snap` siguen los tres contratos y `contrato_1.txt` tiene una sola línea. En `/datos`, dos líneas y sin `contrato_3`.

---

## Parte 3 — Recuperar el archivo y soltar el snapshot

```bash
sudo cp /mnt/snap/empresa/clientes/contrato_3.txt /datos/empresa/clientes/
ls /datos/empresa/clientes/
sudo umount /mnt/snap
sudo lvremove /dev/vg_datos/lv_datos_snap
```
(responder `y`)
```bash
sudo lvs vg_datos
```

**Comprobar:**
```
contrato_1.txt  contrato_2.txt  contrato_3.txt
Do you really want to remove active logical volume vg_datos/lv_datos_snap? [y/n]: y
  Logical volume "lv_datos_snap" successfully removed.
  lv_datos vg_datos -wi-ao---- 2.00g
```
`contrato_3` volvió. `lv_datos` perdió la `o`: ya no tiene snapshot.

---

## Parte 4 — Un volumen con **todo** lo libre, renombrarlo y borrarlo

```bash
sudo lvcreate -n lv_pruebas -l 100%FREE vg_datos
sudo vgs vg_datos
```

Ahora ustedes: renombrarlo a `lv_temporal`, verlo con `lvs`, borrarlo sin que pregunte (`-y`), y ver `vgs`. Foto.

**Comprobar:**
```
  Logical volume "lv_pruebas" created.
  vg_datos   2   2   0 wz--n- 3.99g    0
  Renamed "lv_pruebas" to "lv_temporal" in volume group "vg_datos"
  lv_temporal vg_datos -wi-a----- 1.99g
  Logical volume "lv_temporal" successfully removed
  vg_datos   2   1   0 wz--n- 3.99g 1.99g
```
`VFree` bajó a 0 y volvió a 1.99g. Ese espacio queda para el reto.
