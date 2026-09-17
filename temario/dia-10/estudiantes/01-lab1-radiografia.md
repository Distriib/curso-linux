# Lab 1.1 — Radiografía del servidor

Todos corren cada comando; después lo leemos juntos. El servidor está **sano**: la idea es reconocer la salida normal para distinguir la anormal en el reto. La única falla la vamos a provocar nosotros, en `/datos`.

---

## Parte 1 — ¿Qué falló al arrancar y cuánto tardó?

**¿Cómo sé si algo se rompió al arrancar sin leer todo el log?**

```bash
systemctl --failed
systemd-analyze time
systemd-analyze blame | head -5
```

**Comprobar:**
```
0 loaded units listed.
Startup finished in ...s (kernel) + ...s (initrd) + ...s (userspace) = ...s
multi-user.target reached after ...s in userspace
     ...s NetworkManager-wait-online.service
     ...
```
`0 loaded units listed` es lo que queremos. `NetworkManager-wait-online` primero en `blame` es normal.

---

## Parte 2 — ¿Hay errores de este arranque o del kernel?

**¿Dónde miro si un servicio "se murió solo a las 3 de la mañana"?**

```bash
journalctl -p err -b --no-pager | tail -5
sudo dmesg -T | tail -5
sudo dmesg -T | grep -i oom
```

**Comprobar:** pocas líneas o ninguna en las dos primeras. El `grep -i oom` **no devuelve nada**: el kernel no mató ningún proceso por falta de memoria.

---

## Parte 3 — ¿Qué cambió y quién estuvo?

**¿Alguien instaló algo o entró antes de que dejara de funcionar?**

```bash
w
last -n 5
sudo lastb | head -5
sudo dnf history | head -5
```

**Comprobar:**
```
 ...  up ... min,  1 user,  load average: 0.0x, 0.0x, 0.0x
student  pts/0    10.0.2.2  ...
student  pts/0        10.0.2.2         ... still logged in
reboot   system boot  5.14.0-...       ... still running
btmp begins ...
ID     | Command line              | Date and time    | Action(s)      | Altered
```
`reboot system boot` en `last` es cada vez que arrancó la VM.

---

## Parte 4 — ¿Qué escucha, en qué IP, y el firewall qué deja pasar?

**¿El puerto 80 está abierto? ¿Quién lo tiene?**

```bash
ip -br a
ip route
sudo ss -tlnp
sudo firewall-cmd --get-active-zones
```

**Comprobar:**
```
enp0s3           UP             10.0.2.15/24 ...
enp0s8           UP             192.168.56.10/24 ...
default via 10.0.2.2 dev enp0s3 ...
LISTEN 0  128   0.0.0.0:22   0.0.0.0:*  users:(("sshd",pid=...
LISTEN 0  511   *:80         *:*        users:(("httpd",pid=...
internal
  sources: 192.168.56.0/24
public
  interfaces: enp0s3 enp0s8
```
Si el puerto **no** aparece en `ss`, el firewall todavía no es el problema: el servicio no está escuchando.

---

## Parte 5 — ¿Cómo está la carga?

```bash
uptime
free -m
vmstat 1 3
top -b -n 1 | head -12
```

**Comprobar:** `load average` cerca de `0`. En `free -m`, la columna `available` grande. En `vmstat`, `si` y `so` en `0` (no usa swap) e `id` cerca de `100` (CPU libre).

---

## Parte 6 — El disco lleno que `du` no explica

**¿Qué pasa si borro un archivo grande que un programa tiene abierto?**

```bash
df -h /datos
sudo fallocate -l 300M /datos/temporal.img
df -h /datos
sudo tail -f /datos/temporal.img > /dev/null &
sudo rm /datos/temporal.img
df -h /datos
sudo du -sh /datos
```

**Comprobar:**
```
/dev/mapper/vg_datos-lv_datos  1.5G  ...M  ...  ..% /datos    <- antes
/dev/mapper/vg_datos-lv_datos  1.5G  ...M  ...  ..% /datos    <- 300M más
[1] NNNN
/dev/mapper/vg_datos-lv_datos  1.5G  ...M  ...  ..% /datos    <- borrado, y sigue ocupado
...M	/datos                                                 <- du no lo ve
```
`df` dice que los 300M siguen ocupados; `du` no los encuentra. Es la causa número uno de "borré el log gigante y el disco sigue lleno".

---

## Parte 7 — Encontrar el archivo fantasma y liberar el espacio

**¿Qué proceso tiene abierto un archivo que ya no existe?**

```bash
sudo dnf install -y lsof
sudo lsof +L1
```

**Comprobar:**
```
COMMAND  PID   USER  FD   TYPE  DEVICE  SIZE/OFF   NLINK  NODE  NAME
tail     NNNN  root  3r   REG   253,2   314572800  0      ...   /datos/temporal.img (deleted)
```
Anotar el `PID` y el número del `FD` (el `3` de `3r`).

**¿Cómo libero el espacio sin reiniciar nada?**

```bash
sudo truncate -s 0 /proc/PID/fd/3
df -h /datos
sudo kill PID
jobs
```
(en los dos comandos, cambiar `PID` por el número que mostró `lsof`)

**Comprobar:** `df -h /datos` vuelve al valor de la Parte 6. `jobs` muestra el trabajo `[1]` terminado (`Terminado`, `Done` o `Exit`), ya no corriendo.

Regla: los logs no se borran con `rm`; se vacían con `truncate -s 0` o se rotan con `logrotate` (Lab 2.3).
