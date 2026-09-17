# Lab — Radiografía del servidor (comandos)

Guiado: vos tipeás, ellos tipean, se lee la salida juntos. Antes de la Parte 6, que todos confirmen `findmnt /datos`: quien no lo tenga montado usa `/tmp` en vez de `/datos` en las Partes 6 y 7 (el `df -h /tmp` muestra la raíz, y funciona igual).

## Parte 1
```bash
systemctl --failed
systemd-analyze time
systemd-analyze blame | head -5
```

## Parte 2
```bash
journalctl -p err -b --no-pager | tail -5
sudo dmesg -T | tail -5
sudo dmesg -T | grep -i oom
```
Que no se asusten con el `grep` vacío: vacío es la respuesta buena.

## Parte 3
```bash
w
last -n 5
sudo lastb | head -5
sudo dnf history | head -5
```

## Parte 4
```bash
ip -br a
ip route
sudo ss -tlnp
sudo firewall-cmd --get-active-zones
```
En tu VM (UTM) las interfaces se llaman `enp0s1`/`enp0s2` y la red host-only puede no ser `192.168.56.x`: decirlo antes de que alguien lo note.

## Parte 5
```bash
uptime
free -m
vmstat 1 3
top -b -n 1 | head -12
```

## Parte 6
```bash
findmnt /datos
df -h /datos
sudo fallocate -l 300M /datos/temporal.img
df -h /datos
sudo tail -f /datos/temporal.img > /dev/null &
sudo rm /datos/temporal.img
df -h /datos
sudo du -sh /datos
```
El `sudo` del `tail` no pide contraseña porque el `fallocate` de arriba la acaba de guardar. Si a alguien le queda el trabajo detenido pidiendo contraseña: `fg`, escribirla, `Ctrl+Z`, `bg`.

## Parte 7
```bash
sudo dnf install -y lsof
sudo lsof +L1
```
Cada uno tiene un PID distinto: que lo lean de **su** salida. El FD es el número antes de la `r` (normalmente `3`).
```bash
sudo truncate -s 0 /proc/PID/fd/3
df -h /datos
sudo kill PID
jobs
```
Si `dnf install` falla por la suscripción y `lsof` ya está instalado (`rpm -q lsof`), seguir sin instalar.
