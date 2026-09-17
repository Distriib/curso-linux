# Lab — Un pool Stratis con snapshot y ampliación (comandos)

Vos en UTM: `vdc1`, `vdc2`. El `dnf install` tarda uno o dos minutos: arrancarlo y mientras repetir la idea de thin provisioning.

## Parte 1
```bash
sudo dnf install -y stratis-cli stratisd
sudo systemctl enable --now stratisd
systemctl is-active stratisd
```
Si `dnf` dice `This system is not registered` o no encuentra repositorios: `sudo subscription-manager register` (Día 5).

## Parte 2
```bash
lsblk -f /dev/sdc1
sudo stratis pool create pool1 /dev/sdc1
sudo stratis pool list
```
Si `pool create` dice `has an existing signature` o `is already in use`: `sudo wipefs -a /dev/sdc1` y repetir. Si alguien puso `/dev/sdc` en vez de `/dev/sdc1`: falla porque el disco tiene particiones; corregir a `sdc1`. Si dice `Failed to connect to stratisd`: el servicio no arrancó; repetir el `enable --now`.

## Parte 3
```bash
sudo stratis filesystem create pool1 fs1
sudo stratis filesystem list
sudo mkdir /stratis
sudo mount /dev/stratis/pool1/fs1 /stratis
df -h /stratis
```
Preguntar: *"¿qué pasa si alguien copia 4 GiB acá?"* Respuesta: se llena el pool de 3 GiB y las escrituras fallan, aunque `df` diga que sobran 1000 GiB.

## Parte 4
```bash
echo "archivo en stratis" | sudo tee /stratis/hola.txt
sudo blkid -s UUID -o value /dev/stratis/pool1/fs1
sudo vim /etc/fstab
```
`G` → `o` → pegar/escribir → `Esc` → `:wq`:
```
UUID=2f1d0c9b-...  /stratis  xfs  defaults,x-systemd.requires=stratisd.service  0 0
```
```bash
sudo systemctl daemon-reload
sudo umount /stratis
sudo mount -a
findmnt /stratis
```
Ellos: la misma línea con su UUID. Revisar en las fotos que esté `x-systemd.requires=stratisd.service`: sin eso, el reto (que reinicia) los manda al modo de emergencia.

## Parte 5
```bash
sudo stratis filesystem snapshot pool1 fs1 snap1
sudo stratis filesystem list
sudo rm /stratis/hola.txt
sudo mkdir -p /mnt/snap1
sudo mount /dev/stratis/pool1/snap1 /mnt/snap1
cat /mnt/snap1/hola.txt
sudo umount /mnt/snap1
```
Si el `mount` del snapshot se quejara de UUID duplicado: `sudo mount -o nouuid /dev/stratis/pool1/snap1 /mnt/snap1`.

## Parte 6
```bash
sudo stratis pool add-data pool1 /dev/sdc2
sudo stratis pool list
sudo stratis blockdev list pool1
```
Decir: *"3 GiB a 5 GiB, montado, un comando. Compárenlo con el Lab 3.2."*
