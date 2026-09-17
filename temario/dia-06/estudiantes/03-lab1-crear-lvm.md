# Lab 3.1 — Crear un volumen LVM y montarlo

Vamos a construir `vg_datos` sobre `sdb3`, cortar `lv_datos` de 1 GiB con XFS y dejarlo montado en `/datos` de forma permanente. Todos tipean cada comando; cuando dice **Ahora ustedes**, lo hacen solos. Foto de cada parte. En UTM: `vdb3`.

| Capa | Nombre | Tamaño |
|---|---|---|
| PV | `/dev/sdb3` | 2 GiB |
| VG | `vg_datos` | `<2.00g` |
| LV | `lv_datos` | 1 GiB, XFS, en `/datos` |

---

## Parte 1 — ¿Cómo le entrego una partición a LVM?

```bash
sudo pvcreate /dev/sdb3
sudo pvs
```

**Comprobar:**
```
  Physical volume "/dev/sdb3" successfully created.
  PV         VG   Fmt  Attr PSize   PFree
  /dev/sda2  rhel lvm2 a--  <19.00g     0
  /dev/sdb3       lvm2 ---    2.00g  2.00g
```
Dos PV: `sda2` ya era del VG `rhel` (el instalador). `sdb3` todavía no tiene VG.

---

## Parte 2 — ¿Cómo armo el grupo? ¿Por qué dice `<2.00g`?

```bash
sudo vgcreate vg_datos /dev/sdb3
sudo vgs
sudo vgdisplay vg_datos
```

**Comprobar:**
```
  Volume group "vg_datos" successfully created
  VG       #PV #LV #SN Attr   VSize   VFree
  rhel       1   2   0 wz--n- <19.00g     0
  vg_datos   1   0   0 wz--n-  <2.00g <2.00g
```
```
  PE Size               4.00 MiB
  Total PE              511
  Free  PE / Size       511 / <2.00 GiB
```
511 extents × 4 MiB = 2044 MiB: LVM se quedó 1 MiB para su información. Por eso `<2.00g` y por eso acá no se puede pedir `-L 2G`.

---

## Parte 3 — ¿Cómo corto un volumen de 1 GiB?

```bash
sudo lvcreate -n lv_datos -L 1G vg_datos
sudo lvs
ls -l /dev/vg_datos/lv_datos /dev/mapper/vg_datos-lv_datos
```

**Comprobar:**
```
  Logical volume "lv_datos" created.
  LV       VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root     rhel     -wi-ao---- <17.00g
  swap     rhel     -wi-ao----   2.00g
  lv_datos vg_datos -wi-a-----   1.00g
lrwxrwxrwx. 1 root root 7 ... /dev/mapper/vg_datos-lv_datos -> ../dm-2
lrwxrwxrwx. 1 root root 7 ... /dev/vg_datos/lv_datos -> ../dm-2
```
`lv_datos` es `-wi-a-----`: activo, pero sin la `o`: todavía no está montado. Los dos nombres apuntan al mismo `dm-2`.

---

## Parte 4 — ¿Cómo lo formateo y lo dejo montado para siempre?

```bash
sudo mkfs.xfs /dev/vg_datos/lv_datos
sudo mkdir /datos
sudo vim /etc/fstab
```
`G` → `o` → escribir (esta línea es igual para todos: no lleva UUID):
```
/dev/mapper/vg_datos-lv_datos  /datos  xfs  defaults  0 0
```
`Esc` → `:wq`.
```bash
sudo systemctl daemon-reload
sudo mount -a
df -h /datos
```

**Comprobar:**
```
/dev/mapper/vg_datos-lv_datos 1014M   40M  975M   4% /datos
```
Montado por `fstab`, por nombre y no por UUID: los nombres LVM no cambian entre arranques.

---

## Parte 5 — Datos de prueba

```bash
sudo mkdir -p /datos/empresa/clientes /datos/empresa/logs
echo "Contrato 1 - PanamaTech" | sudo tee /datos/empresa/clientes/contrato_1.txt
```

Ahora ustedes: `contrato_2.txt` y `contrato_3.txt`, cada uno con su número. Foto.

**Comprobar:**
```bash
ls /datos/empresa/clientes/
```
```
contrato_1.txt  contrato_2.txt  contrato_3.txt
```

---

## Parte 6 — Un archivo grande: ¿se nota en `df`?

```bash
sudo dd if=/dev/zero of=/datos/empresa/logs/app.log bs=1M count=100 status=none
sudo du -sh /datos/empresa
df -h /datos
```

**Comprobar:**
```
101M    /datos/empresa
/dev/mapper/vg_datos-lv_datos 1014M  141M  874M  14% /datos
```
Un log de 100 MiB. `df` pasó de 40M a 141M ocupados.

---

## Parte 7 — ¿Cómo quedó, visto desde el disco?

```bash
lsblk -f /dev/sdb
```

**Comprobar:**
```
NAME                  FSTYPE      ...  MOUNTPOINTS
sdb
├─sdb1                xfs         ...  /archivos
├─sdb2                swap        ...  [SWAP]
├─sdb3                LVM2_member
│ └─vg_datos-lv_datos xfs         ...  /datos
└─sdb4
```
`sdb3` dice `LVM2_member` y de él cuelga `vg_datos-lv_datos`. `sdb4` sigue vacío: es para el próximo lab. Foto.
