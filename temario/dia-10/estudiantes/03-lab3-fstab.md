# Lab 3.3 — El servidor no arranca: reparar `/etc/fstab`

Objetivo: provocar el fallo de arranque más común de un servidor (una línea inválida en `fstab` sin `nofail`), reconocer el **emergency mode**, encontrar la causa en el log y corregirla. Las Partes 2 a 4 van **en la ventana de la VM**.

| Qué | Valor |
|---|---|
| Línea falsa | `UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0` |
| Punto de montaje | `/mnt/auditoria` |

---

## Parte 1 — Provocar el fallo

**¿Qué avisos ignora un administrador justo antes de romper el arranque?**

```bash
findmnt --verify
echo "UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo mkdir -p /mnt/auditoria
findmnt --verify
sudo mount -a
```

**Comprobar:**
```
Success, no errors or warnings detected
UUID=deadbeef-0000-4000-8000-00000000c0de  /mnt/auditoria  xfs  defaults  0 0
/mnt/auditoria
   [W] unreachable on boot required source: UUID=deadbeef-0000-4000-8000-00000000c0de
0 parse errors, 0 errors, 1 warning
mount: /mnt/auditoria: can't find UUID=deadbeef-0000-4000-8000-00000000c0de.
```
`findmnt --verify` y `mount -a` **ya avisan**. En la vida real, esta es la comprobación que evita el ticket. Hoy la ignoramos a propósito.

---

## Parte 2 — Reiniciar y mirar la consola

```bash
sudo systemctl reboot
```

**Comprobar** (en la ventana de la VM, después de un minuto y medio de espera):
```
[  TIME ] Timed out waiting for device /dev/disk/by-uuid/deadbeef-0000-4000-8000-00000000c0de.
[DEPEND] Dependency failed for /mnt/auditoria.
[DEPEND] Dependency failed for Local File Systems.
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```
SSH no funciona: la red no arrancó. Escribir la contraseña de root en la consola.

---

## Parte 3 — Diagnosticar

**¿Qué se rompió? El log lo dice.**

```bash
journalctl -xb -p err --no-pager | grep -i depend
systemctl --failed
```

**Comprobar:**
```
... systemd[1]: Dependency failed for /mnt/auditoria.
... systemd[1]: Dependency failed for Local File Systems.
  UNIT                 LOAD   ACTIVE SUB    DESCRIPTION
● mnt-auditoria.mount  loaded failed failed /mnt/auditoria
```
El nombre de la unidad fallida, `mnt-auditoria.mount`, apunta directo a la línea de `fstab`.

---

## Parte 4 — Corregir y seguir arrancando

**¿Cómo edito `fstab` si el disco puede estar en solo lectura?**

```bash
mount -o remount,rw /
vi /etc/fstab
```
Ir a la última línea con `G`, borrarla con `dd`, guardar con `:wq`.

```bash
tail -2 /etc/fstab
systemctl daemon-reload
mount -a
findmnt --verify
systemctl default
```

**Comprobar:** `tail` ya no muestra la línea `deadbeef`. `mount -a` no dice nada. `findmnt --verify` dice `Success, no errors or warnings detected`. `systemctl default` continúa el arranque hasta la pantalla de login normal.

---

## Parte 5 — Verificar por SSH y limpiar

```bash
uptime
findmnt --verify
sudo rmdir /mnt/auditoria
systemctl --failed
```

**Comprobar:** `up` hace pocos minutos; `Success, no errors or warnings detected`; `0 loaded units listed.`
