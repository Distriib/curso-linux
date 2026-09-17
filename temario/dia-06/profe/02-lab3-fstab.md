# Lab — Montaje permanente con `/etc/fstab` (comandos)

Tener la contraseña de root a mano por si alguien reinicia por su cuenta con una línea mala. Nadie reinicia en este lab.

## Parte 1
```bash
sudo blkid -s UUID -o value /dev/sdb1
```
Decir: *"seleccionen el UUID con el mouse; lo van a pegar en `vim`"*. Si alguien no puede copiar y pegar, la alternativa es `LABEL=ARCHIVOS` en vez de `UUID=...`: la etiqueta la pusimos en el lab anterior y también vale.

## Parte 2
```bash
sudo vim /etc/fstab
```
`G` → `o` → pegar/escribir → `Esc` → `:wq`:
```
UUID=c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90  /archivos  xfs  defaults,nofail  0 0
```
```bash
tail -1 /etc/fstab
```
Ellos: la misma línea con su UUID. Mirar en las fotos que no falte `nofail`: es lo que hace que un error no tumbe el arranque.

## Parte 3
```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /archivos
sudo findmnt --verify
```
Si `mount -a` dice `can't find UUID=`: UUID mal copiado. `sudo blkid /dev/sdb1`, `sudo vim /etc/fstab`, corregir, repetir los tres comandos. Si dice `mount point does not exist`: falta `/archivos` (no debería: lo crearon en el Lab 2.2).

## Parte 4
```bash
cd /archivos
sudo umount /archivos
sudo fuser -vm /archivos
```
```bash
cd ~
sudo umount /archivos
sudo mount -a
df -h /archivos
```
Decir: *"`target is busy` casi siempre es uno mismo parado adentro, o un `less` o `tail -f` abierto en otra terminal."*
