# 3 — LVM: volúmenes que crecen

## Tres capas

```
 /dev/sdb3 (2 GiB) ──► PV ──┐
                            ├──► VG vg_datos (3.99 GiB) ──► LV lv_datos (2 GiB) ──► XFS  ──► /datos
 /dev/sdb4 (2 GiB) ──► PV ──┘                              LV lv_backups        ──► ext4 ──► /backups
```

| Capa | Crear | Ver (resumen) | Ver (detalle) | Ampliar | Borrar |
|---|---|---|---|---|---|
| PV — volumen físico | `pvcreate /dev/sdb3` | `pvs` | `pvdisplay` | — | `pvremove` |
| VG — grupo de volúmenes | `vgcreate vg_datos /dev/sdb3` | `vgs` | `vgdisplay` | `vgextend vg_datos /dev/sdb4` | `vgremove` |
| LV — volumen lógico | `lvcreate -n lv_datos -L 1G vg_datos` | `lvs` | `lvdisplay` | `lvextend -L +512M -r` | `lvremove` |

Todos con `sudo`. La letra de la capa + `s` = resumen; + `display` = detalle.

```bash
sudo pvs
sudo vgs
sudo lvs
```
El sistema ya tiene: PV `/dev/sda2`, VG `rhel`, LV `root` y `swap`.

## Anatomía

```
lvcreate  -n lv_datos  -L 1G  vg_datos
              │           │      └ de qué grupo sacar el espacio
              │           └ tamaño: 1G, 512M; +512M = agregar a lo que tiene
              └ nombre
```

| Tamaño | Significa |
|---|---|
| `-L 1G` | 1 GiB exacto |
| `-L +512M` | 512 MiB **más** de lo que tiene |
| `-l 100%FREE` | todo lo que queda libre en el VG (ele minúscula) |
| `-l 50%VG` | la mitad del VG |

El VG reparte el espacio en **extents** (PE) de 4 MiB, y se queda 1 MiB para su propia información. Por eso una partición de 2 GiB da un VG de `<2.00g`: pedir `-L 2G` ahí falla por unos pocos extents; se pide `-L 1G` o `-l 100%FREE`.

## Dónde aparece un LV

```
/dev/vg_datos/lv_datos          ─┐
                                 ├─► los dos son el mismo /dev/dm-2
/dev/mapper/vg_datos-lv_datos   ─┘   este es el que va en /etc/fstab
```
Los nombres LVM son estables: en `fstab` se puede poner `/dev/mapper/vg_datos-lv_datos` en vez de un UUID.

## Ampliar en caliente

```bash
sudo vgextend vg_datos /dev/sdb4
sudo lvextend -L +512M -r /dev/vg_datos/lv_datos
```

`-r` = redimensionar también el sistema de archivos. **Sin `-r`, el LV crece pero `df -h` sigue igual**: es el error número uno. Se corrige con:

| Sistema de archivos | Comando |
|---|---|
| XFS | `sudo xfs_growfs /datos` (montado) |
| ext4 | `sudo resize2fs /dev/vg_datos/lv_backups` |

## Leer `lvs`

```
  LV       VG       Attr       LSize
  lv_datos vg_datos -wi-ao---- 1.00g
                    ││││││
                    │││││└ o = abierto (montado)
                    ││││└ a = activo
                    │││└ (sin uso hoy)
                    ││└ i = asignación heredada
                    │└ w = escritura
                    └ tipo: - normal · o = origen de un snapshot · s = snapshot
```

## Snapshot

```bash
sudo lvcreate -s -n lv_datos_snap -L 200M /dev/vg_datos/lv_datos
sudo mount -o nouuid /dev/vg_datos/lv_datos_snap /mnt/snap
```
Una foto del volumen en ese instante. Guarda solo los bloques que cambian después: por eso puede ser chico; si se llena, se invalida. El snapshot de un XFS tiene el **mismo UUID** que el original: se monta con `nouuid`. No es un respaldo: vive en el mismo disco.

## Renombrar, borrar, y el orden de destrucción

| Comando | Qué hace |
|---|---|
| `sudo lvrename vg_datos lv_pruebas lv_temporal` | renombrar un LV (grupo, nombre viejo, nombre nuevo) |
| `sudo lvremove /dev/vg_datos/lv_temporal` | borrar (pregunta; con `-y` no pregunta) |
| `sudo vgrename vg_datos vg_produccion` | renombrar un VG (y después corregir `fstab`) |

Para deshacer todo, al revés de como se armó: `umount` → `lvremove` → `vgremove` → `pvremove`.
