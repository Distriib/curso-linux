# Lab 4.1 — Un pool Stratis con snapshot y ampliación

Vamos a instalar Stratis, crear `pool1` sobre `sdc1`, un sistema de archivos `fs1` montado permanente en `/stratis`, sacarle un snapshot y ampliar el pool con `sdc2`. Foto de cada parte. Donde dice **Ahora ustedes**, el comando no está escrito: hay que resolverlo. La solución está al final de la hoja. En UTM: `vdc1`, `vdc2`.

---

## Parte 1 — Instalar y arrancar el servicio

```bash
sudo dnf install -y stratis-cli stratisd
sudo systemctl enable --now stratisd
systemctl is-active stratisd
```

**Comprobar:**
```
Complete!
Created symlink /etc/systemd/system/multi-user.target.wants/stratisd.service → /usr/lib/systemd/system/stratisd.service.
active
```
Si `dnf` dice que no hay repositorios, la VM no está registrada: `sudo subscription-manager register`.

---

## Parte 2 — ¿Está limpio `sdc1`? Crear el pool

```bash
lsblk -f /dev/sdc1
sudo stratis pool create pool1 /dev/sdc1
sudo stratis pool list
```

**Comprobar:**
```
pool1  3 GiB / 37.63 MiB / 2.96 GiB   ~Ca,~Cr, Op   7c2e9b1a-...
```
`lsblk -f` no muestra `FSTYPE`: la partición está limpia. Si mostrara algo: `sudo wipefs -a /dev/sdc1` y repetir. Las tres cifras son total, ocupado y libre: el pool ya ocupa unas decenas de MiB sin un solo archivo, con su propia información.

---

## Parte 3 — Crear el sistema de archivos y montarlo: ¿cuánto mide?

```bash
sudo stratis filesystem create pool1 fs1
sudo stratis filesystem list
sudo mkdir /stratis
sudo mount /dev/stratis/pool1/fs1 /stratis
df -h /stratis
```

**Comprobar:**
```
pool1  fs1   1 TiB / 546 MiB / 1023.47 GiB   ...   /dev/stratis/pool1/fs1   2f1d0c9b-...
/dev/mapper/stratis-1-7c2e...-thin-fs-2f1d...   1.0T  7.2G  1017G   1% /stratis
```
`df` dice **1 TiB** sobre una partición de 3 GiB. Es virtual: el espacio real lo dice `stratis pool list`.

---

## Parte 4 — Escribir algo y dejarlo permanente

```bash
echo "archivo en stratis" | sudo tee /stratis/hola.txt
sudo blkid -s UUID -o value /dev/stratis/pool1/fs1
sudo vim /etc/fstab
```
`G` → `o` → escribir, pegando el UUID:
```
UUID=2f1d0c9b-...  /stratis  xfs  defaults,x-systemd.requires=stratisd.service  0 0
```
`Esc` → `:wq`.
```bash
sudo systemctl daemon-reload
sudo umount /stratis
sudo mount -a
findmnt /stratis
```

Ahora ustedes: la misma línea, con **su** UUID. Foto del `findmnt`.

**Comprobar:**
```
TARGET   SOURCE                                        FSTYPE OPTIONS
/stratis /dev/mapper/stratis-1-...-thin-fs-2f1d...     xfs    rw,relatime,seclabel,...
```
Lo montó `mount -a` desde `fstab`. Sin `x-systemd.requires=stratisd.service` en las opciones, en el próximo arranque la VM caería en modo de emergencia.

---

## Parte 5 — Snapshot: borro el archivo, ¿lo recupero?

```bash
sudo stratis filesystem snapshot pool1 fs1 snap1
sudo stratis filesystem list
sudo rm /stratis/hola.txt
sudo mkdir -p /mnt/snap1
sudo mount /dev/stratis/pool1/snap1 /mnt/snap1
cat /mnt/snap1/hola.txt
sudo umount /mnt/snap1
```

**Comprobar:**
```
pool1  fs1     1 TiB / 546 MiB / ...   ...   /dev/stratis/pool1/fs1     2f1d...
pool1  snap1   1 TiB / 546 MiB / ...   ...   /dev/stratis/pool1/snap1   8a3b...
archivo en stratis
```
El snapshot es un sistema de archivos más, con su propio UUID (por eso acá no hace falta `nouuid`). El archivo borrado sigue en `/mnt/snap1`.

---

## Parte 6 — Ampliar el pool en caliente

```bash
sudo stratis pool add-data pool1 /dev/sdc2
sudo stratis pool list
sudo stratis blockdev list pool1
```

**Comprobar:**
```
pool1  5 GiB / 548 MiB / 4.46 GiB   ~Ca,~Cr, Op   7c2e...
Pool Name   Device Node   Physical Size   Tier   UUID
pool1       /dev/sdc1     3 GiB           Data   ...
pool1       /dev/sdc2     2 GiB           Data   ...
```
El pool pasó de 3 a 5 GiB con `/stratis` montado. Un solo comando: lo que en LVM fue `pvcreate` + `vgextend` + `lvextend -r`. Foto.

---

# Solución — todos los comandos

```bash
# Parte 1 — instalar y arrancar
sudo dnf install -y stratis-cli stratisd
sudo systemctl enable --now stratisd
systemctl is-active stratisd

# Parte 2 — el pool
lsblk -f /dev/sdc1
sudo stratis pool create pool1 /dev/sdc1
sudo stratis pool list

# Parte 3 — el sistema de archivos
sudo stratis filesystem create pool1 fs1
sudo stratis filesystem list
sudo mkdir /stratis
sudo mount /dev/stratis/pool1/fs1 /stratis
df -h /stratis

# Parte 4 — escribir algo y sacar el UUID
echo "archivo en stratis" | sudo tee /stratis/hola.txt
sudo blkid -s UUID -o value /dev/stratis/pool1/fs1
```

**Parte 4 (Ahora ustedes)** — la línea de `fstab` con **su propio** UUID:
```bash
sudo vim /etc/fstab
```
`G` → `o` → pegar, reemplazando el UUID:
```
UUID=2f1d0c9b-...  /stratis  xfs  defaults,x-systemd.requires=stratisd.service  0 0
```
`Esc` → `:wq`
```bash
sudo systemctl daemon-reload
sudo umount /stratis
sudo mount -a
findmnt /stratis
```
Sin `x-systemd.requires=stratisd.service`, en el próximo arranque el dispositivo todavía no existe y la VM cae en modo de emergencia.

```bash
# Parte 5 — snapshot
sudo stratis filesystem snapshot pool1 fs1 snap1
sudo stratis filesystem list
sudo rm /stratis/hola.txt
sudo mkdir -p /mnt/snap1
sudo mount /dev/stratis/pool1/snap1 /mnt/snap1
cat /mnt/snap1/hola.txt
sudo umount /mnt/snap1

# Parte 6 — ampliar el pool en caliente
sudo stratis pool add-data pool1 /dev/sdc2
sudo stratis pool list
sudo stratis blockdev list pool1
```
Si `lsblk -f` mostrara algo en `sdc1`, limpiarlo con `sudo wipefs -a /dev/sdc1` y repetir. En UTM, `vdc1` y `vdc2`.
