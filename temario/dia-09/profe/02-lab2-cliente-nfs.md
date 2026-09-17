# Lab — Cliente NFS y `fstab` (comandos)

## Parte 1 — Montar a mano
```bash
sudo mkdir -p /mnt/nfs /mnt/lectura
sudo mount -t nfs 192.168.56.10:/srv/nfs/compartido /mnt/nfs
sudo mount -t nfs 127.0.0.1:/srv/nfs/lectura /mnt/lectura
mount | grep nfs4
df -hT /mnt/nfs /mnt/lectura
```
Decir: "`vers=4.2` sin pedirlo. Y el `rw` de `/mnt/lectura` es lo que pidió el cliente: el servidor lo exportó `ro` y el servidor manda."

Si a alguien le da `access denied by server`: su IP host-only no está en la red del export (mirar `ip -4 addr`), o no corrió `exportfs -rav`.

## Parte 2 — Escribir, leer, y chocar
```bash
echo "escrito desde el cliente" > /mnt/nfs/prueba.txt
cat /mnt/nfs/prueba.txt
ls -l /srv/nfs/compartido/
touch /mnt/lectura/no-se-puede.txt
sudo touch /mnt/lectura/tampoco-root.txt
```
Decir: "mismo dueño en los dos lados porque el UID coincide. NFS trabaja con números, como el Día 3."

## Parte 3 — La etiqueta SELinux del cliente
```bash
ls -Zd /mnt/nfs
```

## Parte 4 — Permanente con `fstab`
```bash
sudo umount /mnt/nfs /mnt/lectura
echo "192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/nfs
```
Decir: "`_netdev` = esperá a la red; `nofail` = si no podés, seguí arrancando. Y `mount -a` antes de reiniciar, siempre: un error en `fstab` se ve ahora y no en modo emergencia."

## Parte 5 — Dejar la línea comentada
```bash
sudo umount /mnt/nfs
sudo vim /etc/fstab
```
`G` (última línea), `I`, escribir `#`, `Esc`, `:wq`.
```bash
tail -1 /etc/fstab
sudo systemctl daemon-reload
```
Decir: "en un cliente real la línea se deja. Acá la comentamos porque el próximo lab monta lo mismo con autofs y no queremos dos mecanismos peleando."
