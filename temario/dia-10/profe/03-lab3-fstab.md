# Lab — El servidor no arranca: reparar `/etc/fstab` (comandos)

Guiado. Parte 1 por SSH; Partes 2 a 4 en la ventana de la VM; Parte 5 otra vez por SSH.

## Parte 1
```bash
findmnt --verify
echo "UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo mkdir -p /mnt/auditoria
findmnt --verify
sudo mount -a
```
**La frase:** "estos dos avisos son los que un administrador ignora cinco minutos antes de romper el arranque. Hoy los ignoramos a propósito."

## Parte 2
```bash
sudo systemctl reboot
```
Espera de ~90 segundos (systemd espera el disco antes de rendirse). Contraseña de root en la consola.

## Parte 3
```bash
journalctl -xb -p err --no-pager | grep -i depend
systemctl --failed
```

## Parte 4
```bash
mount -o remount,rw /
vi /etc/fstab
```
`G` (última línea), `dd` (borrarla), `:wq`. Si `vi` avisa `Read-only file system`, faltó el `remount,rw`.
```bash
tail -2 /etc/fstab
systemctl daemon-reload
mount -a
findmnt --verify
systemctl default
```
Si alguien hace `systemctl default` y vuelve a caer en emergency: no borró la línea correcta, o le faltó `daemon-reload`. Repetir desde `vi`.
`findmnt --verify` limpio dice `Success, no errors or warnings detected`; si en alguna VM dice `0 parse errors, 0 errors, 0 warnings`, es lo mismo.

## Parte 5
```bash
uptime
findmnt --verify
sudo rmdir /mnt/auditoria
systemctl --failed
```
