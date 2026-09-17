# Lab — Swap en partición (comandos)

Vos en UTM: `vdb2`.

## Parte 1
```bash
swapon --show
free -m
```

## Parte 2
```bash
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
free -m
```
Señalar `PRIO -2` y `-3`: el kernel usa primero la de número más alto.

## Parte 3
```bash
sudo blkid -s UUID -o value /dev/sdb2
sudo vim /etc/fstab
```
`G` → `o` → pegar/escribir → `Esc` → `:wq`:
```
UUID=9e1f2a3b-4c5d-6e7f-8a9b-0c1d2e3f4a5b  swap  swap  defaults  0 0
```
```bash
sudo systemctl daemon-reload
tail -2 /etc/fstab
```
Ellos: la misma línea con su UUID. Sin `nofail`: en swap no aplica.

## Parte 4
```bash
sudo swapoff /dev/sdb2
swapon --show
sudo swapon -a
swapon --show
```
Si después de `swapon -a` no vuelve `/dev/sdb2`: la línea de `fstab` está mal (UUID o campos); `sudo swapon -a` muestra el error.
