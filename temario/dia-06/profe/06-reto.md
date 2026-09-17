# Reto — Solución

Ticket #2026-0614. El VG tiene 1.99g libres al terminar el Lab 3.3: alcanza para 1 GiB + 500 MiB.

```bash
# 1. Volumen lv_backups, ext4, permanente, con nofail
sudo lvcreate -n lv_backups -L 1G vg_datos
sudo mkfs.ext4 /dev/vg_datos/lv_backups
sudo mkdir /backups
sudo vim /etc/fstab
```
Línea a agregar (`G`, `o`, escribir, `Esc`, `:wq`):
```
/dev/mapper/vg_datos-lv_backups  /backups  ext4  defaults,nofail  0 0
```
```bash
sudo systemctl daemon-reload
sudo mount -a
sudo findmnt --verify
df -h /backups
```
También vale `UUID=...` en el campo 1 (el UUID lo da `sudo blkid -s UUID -o value /dev/vg_datos/lv_backups`). En el campo 6 vale `0` o `2`.

```bash
# 2. Archivo swap de 512 MiB, permanente
sudo dd if=/dev/zero of=/swapfile bs=1M count=512 status=progress
sudo chmod 600 /swapfile
sudo restorecon -v /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo vim /etc/fstab
```
Línea a agregar:
```
/swapfile  swap  swap  defaults  0 0
```
```bash
sudo systemctl daemon-reload
swapon --show
```

```bash
# 3. Reiniciar y comprobar
sudo findmnt --verify
sudo reboot
```
Esperar un minuto; reconectar con `ssh -p 2222 student@localhost` (o como cada uno se conecte).
```bash
df -h /backups /datos /archivos /stratis
swapon --show
```

```bash
# 4. Segunda parte: 500 MiB más, en caliente
df -h /backups
sudo lvextend -L +500M -r /dev/vg_datos/lv_backups
df -h /backups
sudo lvs vg_datos
```
Con `-r` sobre ext4, `lvextend` llama a `resize2fs` solo. `+500M` son 125 extents: `lvs` muestra `<1.49g`. Quien haya puesto `-L 1.5G` ve `1.50g`. Las dos cumplen.

## Verificación

```bash
df -h /backups /datos /archivos /stratis
swapon --show
sudo lvs vg_datos
grep backups /etc/fstab
grep swapfile /etc/fstab
sudo findmnt --verify
ls -l /swapfile
```

Lo que importa de la salida:
```
/dev/mapper/vg_datos-lv_backups  1.5G   24K  1.4G   1% /backups
/dev/dm-1  partition   2G   0B   -2
/dev/sdb2  partition 512M   0B   -3
/swapfile  file      512M   0B   -4
  lv_backups vg_datos -wi-ao---- <1.49g
  lv_datos   vg_datos -wi-ao----  2.00g
/dev/mapper/vg_datos-lv_backups  /backups  ext4  defaults,nofail  0 0
/swapfile  swap  swap  defaults  0 0
Success, no errors or warnings detected
-rw-------. 1 root root 536870912 ... /swapfile
```

## Errores que se ven

- Olvidar `-r` en la segunda parte: `lvs` dice 1.49g y `df` sigue en 1G. Que lo detecte y corrija con `sudo resize2fs /dev/vg_datos/lv_backups`.
- Olvidar `nofail`: cumple todo hoy, pero no cumple "el servidor tiene que seguir arrancando si el disco falla". Pedir que lo agregue.
- Olvidar `mkfs.ext4`: `mount -a` dice `wrong fs type, bad option, bad superblock`. Formatear y repetir.
- Olvidar `mkdir /backups`: `mount point does not exist`.
- Olvidar `chmod 600`: `mkswap` avisa `insecure permissions 0644, fix with: chmod 0600 /swapfile`. Hacer el `chmod` y repetir `mkswap`.
- Usar `fallocate` en vez de `dd`: `swapon: /swapfile: skipping - it appears to have holes`. Borrar el archivo y rehacerlo con `dd`.
- Olvidar `daemon-reload`: `mount` muestra el aviso `(hint) your fstab has been modified`. Correrlo y seguir.
- Poner `-L 500M` (sin `+`) en la segunda parte: `lvextend` se niega porque 500M es menos de lo que tiene. Es `-L +500M`.
- La VM no vuelve del reinicio: modo de emergencia por una línea mala en `fstab`, casi siempre la de Stratis sin `x-systemd.requires=stratisd.service` o un UUID mal copiado. Caja de emergencia de `02-comandos`: root en la ventana de la VM, `mount -o remount,rw /`, `vi /etc/fstab`, `#` en la línea, `daemon-reload`, `mount -a`, `systemctl default`.
- Correr `restorecon` antes de que exista el archivo: no hace nada. El orden es `dd`, `chmod`, `restorecon`, `mkswap`, `swapon`.
