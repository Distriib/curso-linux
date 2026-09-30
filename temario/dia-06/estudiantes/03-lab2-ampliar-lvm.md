# Lab 3.2 — Ampliar en caliente

El ticket: "`/datos` se va a quedar sin espacio. Hay que darle 1 GiB más **sin desmontar** y sin ventana de mantenimiento." Foto de cada parte. En este lab van todos los comandos escritos; la solución completa está igual al final de la hoja. En UTM: `vdb4`.

---

## Parte 1 — ¿Cómo le agrego más disco al grupo?

```bash
sudo pvcreate /dev/sdb4
sudo vgextend vg_datos /dev/sdb4
sudo vgs vg_datos
sudo pvs
```

**Comprobar:**
```
  Physical volume "/dev/sdb4" successfully created.
  Volume group "vg_datos" successfully extended
  VG       #PV #LV #SN Attr   VSize VFree
  vg_datos   2   1   0 wz--n- 3.99g 2.99g
  PV         VG       Fmt  Attr PSize   PFree
  /dev/sda2  rhel     lvm2 a--  <19.00g     0
  /dev/sdb3  vg_datos lvm2 a--   <2.00g 1020.00m
  /dev/sdb4  vg_datos lvm2 a--   <2.00g   <2.00g
```
El VG pasó de 2 a 3.99 GiB con `/datos` montado y en uso.

---

## Parte 2 — Ampliar el volumen **olvidando** `-r`: ¿qué pasa?

```bash
sudo lvextend -L +512M /dev/vg_datos/lv_datos
sudo lvs vg_datos
df -h /datos
```

**Comprobar:**
```
  Size of logical volume vg_datos/lv_datos changed from 1.00 GiB (256 extents) to 1.50 GiB (384 extents).
  Logical volume vg_datos/lv_datos successfully resized.
  LV       VG       Attr       LSize
  lv_datos vg_datos -wi-ao---- 1.50g
/dev/mapper/vg_datos-lv_datos 1014M  141M  874M  14% /datos
```
`lvs` dice 1.50g. `df` sigue en 1014M. El recipiente creció; el sistema de archivos no se enteró. Este es el ticket "amplié el disco y sigue lleno".

---

## Parte 3 — ¿Cómo lo arreglo?

```bash
sudo xfs_growfs /datos
df -h /datos
```

**Comprobar:**
```
data blocks changed from 262144 to 393216
/dev/mapper/vg_datos-lv_datos  1.5G  141M  1.4G  10% /datos
```
Ahora `df` también dice 1.5G. XFS crece **montado**.

---

## Parte 4 — Bien hecho, en un solo paso

```bash
sudo lvextend -L +512M -r /dev/vg_datos/lv_datos
df -h /datos
sudo lvs vg_datos
sudo vgs vg_datos
ls /datos/empresa/clientes/
```

**Comprobar:**
```
  Size of logical volume vg_datos/lv_datos changed from 1.50 GiB (384 extents) to 2.00 GiB (512 extents).
  Logical volume vg_datos/lv_datos successfully resized.
data blocks changed from 393216 to 524288
/dev/mapper/vg_datos-lv_datos  2.0G  141M  1.9G   7% /datos
  lv_datos vg_datos -wi-ao---- 2.00g
  vg_datos   2   1   0 wz--n- 3.99g 1.99g
contrato_1.txt  contrato_2.txt  contrato_3.txt
```
Con `-r`, `lvextend` hizo las dos cosas. Los contratos siguen ahí y nadie desmontó nada. Quedan 1.99g libres en el VG: se usan en el reto.

---

## Parte 5 — ¿De qué particiones está hecho `/datos` ahora?

```bash
lsblk -f /dev/sdb
sudo pvs
```

**Comprobar:**
```
├─sdb3                LVM2_member
│ └─vg_datos-lv_datos xfs         ...  /datos
└─sdb4                LVM2_member
  └─vg_datos-lv_datos xfs         ...  /datos
  PV         VG       Fmt  Attr PSize   PFree
  /dev/sdb3  vg_datos lvm2 a--   <2.00g     0
  /dev/sdb4  vg_datos lvm2 a--   <2.00g  1.99g
```
`lv_datos` cuelga de `sdb3` **y** de `sdb4`: sus extents están repartidos en los dos. Eso es lo que una partición común no puede hacer. Foto.

---

# Solución — todos los comandos

```bash
# Parte 1 — sumar sdb4 al grupo
sudo pvcreate /dev/sdb4
sudo vgextend vg_datos /dev/sdb4
sudo vgs vg_datos
sudo pvs

# Parte 2 — ampliar OLVIDANDO -r (a propósito)
sudo lvextend -L +512M /dev/vg_datos/lv_datos
sudo lvs vg_datos
df -h /datos                    # sigue en 1014M: el error número uno

# Parte 3 — arreglarlo
sudo xfs_growfs /datos
df -h /datos                    # ahora sí, 1.5G

# Parte 4 — bien hecho, en un solo paso
sudo lvextend -L +512M -r /dev/vg_datos/lv_datos
df -h /datos
sudo lvs vg_datos
sudo vgs vg_datos
ls /datos/empresa/clientes/

# Parte 5 — de qué particiones está hecho /datos
lsblk -f /dev/sdb
sudo pvs
```
Nada se desmontó en todo el lab. Quedan `1.99g` libres en el VG: se usan en el reto. En UTM, `vdb4` y `vdb`.
