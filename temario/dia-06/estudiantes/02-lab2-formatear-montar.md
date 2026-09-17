# Lab 2.2 — Formatear, etiquetar y montar a mano

Vamos a darle un sistema de archivos a `sdb1`, etiquetarlo `ARCHIVOS`, montarlo en `/archivos`, y ver qué pasa al formatear de nuevo. Todos tipean cada comando; cuando dice **Ahora ustedes**, lo hacen solos. Foto de cada parte. En UTM: `vdb1`.

---

## Parte 1 — ¿Cómo se crea un sistema de archivos XFS?

```bash
sudo mkfs.xfs /dev/sdb1
```

**Comprobar:**
```
meta-data=/dev/sdb1              isize=512    agcount=4, agsize=32768 blks
...
data     =                       bsize=4096   blocks=131072, imaxpct=25
...
log      =internal log           bsize=4096   blocks=2560, version=2
```
`blocks=131072` × `bsize=4096` = 512 MiB. Tardó un segundo: solo escribió la estructura.

---

## Parte 2 — ¿Qué firma quedó y cómo le pongo nombre?

```bash
sudo blkid /dev/sdb1
sudo xfs_admin -L ARCHIVOS /dev/sdb1
sudo blkid /dev/sdb1
```

**Comprobar:**
```
/dev/sdb1: UUID="3f0c2b7e-..." TYPE="xfs" PARTLABEL="primary" PARTUUID="a1b2c3d4-..."
writing all SBs
new label = "ARCHIVOS"
/dev/sdb1: LABEL="ARCHIVOS" UUID="3f0c2b7e-..." TYPE="xfs" PARTLABEL="primary" PARTUUID="a1b2c3d4-..."
```
Cada VM tiene un UUID distinto. `UUID` es del sistema de archivos; `PARTUUID` y `PARTLABEL` son de la partición: los puso `parted`.

---

## Parte 3 — ¿Cómo hago que aparezca en el árbol?

```bash
sudo mkdir /archivos
sudo mount /dev/sdb1 /archivos
findmnt /archivos
df -h /archivos
```

**Comprobar:**
```
TARGET    SOURCE    FSTYPE OPTIONS
/archivos /dev/sdb1 xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
/dev/sdb1       502M   29M  473M   6% /archivos
```
`502M`, no 512: el resto es la estructura interna de XFS.

---

## Parte 4 — ¿Los archivos están en la carpeta o en el disco?

```bash
echo "hola desde sdb1" | sudo tee /archivos/nota.txt
ls -l /archivos
sudo umount /archivos
ls -l /archivos
sudo mount /dev/sdb1 /archivos
ls -l /archivos
```

Ahora ustedes: lo mismo, con una nota que diga su nombre. Foto de los tres `ls`.

**Comprobar:** con el disco montado, `nota.txt` está. Después de `umount`, la carpeta está **vacía** (`total 0`). Al volver a montar, reaparece. Los datos viven en el sistema de archivos, no en la carpeta.

---

## Parte 5 — ¿Y si lo quiero en ext4?

```bash
sudo umount /archivos
sudo mkfs.ext4 /dev/sdb1
```
(responder `y`)
```bash
sudo e2label /dev/sdb1 ARCHIVOS
sudo blkid /dev/sdb1
```

**Comprobar:**
```
/dev/sdb1 contains a xfs file system labelled 'ARCHIVOS'
Proceed anyway? (y,N) y
...
/dev/sdb1: LABEL="ARCHIVOS" UUID="7b9d4e10-..." BLOCK_SIZE="4096" TYPE="ext4" ...
```
`mkfs.ext4` avisó que había un XFS. `nota.txt` ya no existe. El **UUID cambió**.

---

## Parte 6 — Volver a XFS: ¿por qué se niega?

```bash
sudo mkfs.xfs /dev/sdb1
sudo mkfs.xfs -f -L ARCHIVOS /dev/sdb1
sudo blkid /dev/sdb1
```

**Comprobar:**
```
mkfs.xfs: /dev/sdb1 appears to contain an existing filesystem (ext4).
mkfs.xfs: Use the -f option to force overwrite.
...
/dev/sdb1: LABEL="ARCHIVOS" UUID="c81e7a22-..." TYPE="xfs" ...
```
`mkfs.xfs` no pregunta: se niega, y hay que forzar con `-f`. Otro UUID nuevo: **este** es el que va a `fstab`. Queda **desmontado** para el siguiente lab. Foto.
