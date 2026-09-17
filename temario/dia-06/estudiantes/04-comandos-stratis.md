# 4 — Stratis

Una sola herramienta que hace lo que en LVM son varios pasos: `stratis`. Un servicio (`stratisd`) administra **pools** de discos y les corta sistemas de archivos XFS.

| LVM | Stratis |
|---|---|
| PV + VG | pool |
| LV + `mkfs.xfs` | filesystem (ya viene con XFS) |
| `vgextend` | `pool add-data` |
| `lvcreate -s` | `filesystem snapshot` |
| `/dev/mapper/vg-lv` | `/dev/stratis/pool/fs` |

## Comandos

| Comando | Qué hace |
|---|---|
| `sudo dnf install -y stratis-cli stratisd` | instalar |
| `sudo systemctl enable --now stratisd` | arrancar el servicio, ahora y en cada arranque |
| `sudo stratis pool create pool1 /dev/sdc1` | crear el pool con un disco o partición **sin firma** |
| `sudo stratis pool list` | ver pools: total, ocupado, libre |
| `sudo stratis filesystem create pool1 fs1` | crear un sistema de archivos en el pool |
| `sudo stratis filesystem list` | ver sistemas de archivos y su UUID |
| `sudo stratis filesystem snapshot pool1 fs1 snap1` | snapshot |
| `sudo stratis pool add-data pool1 /dev/sdc2` | ampliar el pool |
| `sudo stratis blockdev list pool1` | qué discos tiene el pool |
| `sudo stratis filesystem destroy pool1 snap1` | borrar un sistema de archivos |
| `sudo wipefs -a /dev/sdc1` | limpiar la firma si el disco tuvo algo antes |

## Lo que sorprende: `df` dice 1 TiB

```
/dev/mapper/stratis-1-...-thin-fs-...   1.0T  7.2G  1017G   1% /stratis
```
Cada filesystem se presenta con **1 TiB virtual**, aunque el pool tenga 3 GiB. El espacio real se gasta a medida que se escribe: eso es **aprovisionamiento delgado** (thin provisioning). `df` miente; el que dice la verdad es `stratis pool list`. Si el pool se llena, las escrituras fallan aunque `df` diga que sobra.

## La línea de `fstab`

```
UUID=2f1d0c9b-...  /stratis  xfs  defaults,x-systemd.requires=stratisd.service  0 0
```
`x-systemd.requires=stratisd.service`: esperar a que arranque el servicio antes de montar. Sin eso, al arrancar el dispositivo todavía no existe y la VM cae en modo de emergencia. El UUID es el que muestra `stratis filesystem list` (y `blkid`).
