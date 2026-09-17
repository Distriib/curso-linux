# Lab 3.1 — GRUB visible y kernels

Todos corren cada comando. Objetivo: que el menú de GRUB aparezca **10 segundos** al arrancar (lo necesitamos en los dos labs siguientes), leer la configuración con `grubby`, y quitar `rhgb quiet` para ver el arranque completo.

---

## Parte 1 — Cómo está GRUB hoy

**¿Qué kernel arranca, con qué parámetros, y cuántos kernels hay?**

```bash
cat /etc/default/grub
sudo grub2-editenv list
sudo grubby --default-kernel
sudo grubby --info=DEFAULT
ls /boot/loader/entries/
rpm -q kernel
```

**Comprobar:**
```
GRUB_TIMEOUT=5
...
GRUB_DEFAULT=saved
GRUB_CMDLINE_LINUX="crashkernel=... resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
GRUB_ENABLE_BLSCFG=true
saved_entry=...-5.14.0-...
boot_success=1
/boot/vmlinuz-5.14.0-...
index=0
kernel="/boot/vmlinuz-5.14.0-..."
args="ro crashkernel=... rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
root="/dev/mapper/rhel-root"
initrd="/boot/initramfs-5.14.0-....img"
title="Red Hat Enterprise Linux (5.14.0-...) 9.x (Plow)"
...-5.14.0-....conf  ...-0-rescue.conf
kernel-5.14.0-...
```
Hay una entrada por kernel más la `rescue` (un initramfs con todos los drivers, para cuando el normal no arranca). Si `rpm -q kernel` lista dos, hubo un `dnf update` y GRUB muestra ambos.

---

## Parte 2 — Que el menú se vea 10 segundos

**¿Cómo hago que GRUB espere y muestre el menú?**

```bash
sudo grub2-editenv - unset menu_auto_hide
sudo vim /etc/default/grub
```
Cambiar `GRUB_TIMEOUT=5` por `GRUB_TIMEOUT=10`. Guardar (`:wq`).

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
grep GRUB_TIMEOUT /etc/default/grub
sudo grub2-editenv list
```

**Comprobar:**
```
Generating grub configuration file ...
done
GRUB_TIMEOUT=10
saved_entry=...
boot_success=1
```
`menu_auto_hide` ya no aparece en la lista. En una VM UEFI sale además `Adding boot menu entry for UEFI Firmware Settings ...`.

---

## Parte 3 — Ver el arranque completo

**¿Por qué no veo los mensajes al arrancar?**

```bash
sudo grubby --update-kernel=ALL --remove-args="rhgb quiet"
sudo grubby --info=DEFAULT | grep args
```

**Comprobar:**
```
args="ro crashkernel=... resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap"
```
Sin `rhgb quiet`. `rhgb` es la pantalla gráfica de arranque y `quiet` silencia los mensajes; sin ellos se ve todo lo que hacen el kernel y systemd. Lo necesitamos para ver el `emergency mode` del Lab 3.3.

---

## Parte 4 — Targets

**¿Con qué target arranca el servidor, y qué es `emergency`?**

```bash
systemctl get-default
systemctl list-dependencies rescue.target --no-pager | head -6
systemctl cat emergency.target | grep Requires
```

**Comprobar:**
```
multi-user.target
rescue.target
● ├─rescue.service
● ├─sysinit.target
...
Requires=emergency.service
```
`set-default` solo cambia un enlace: `/etc/systemd/system/default.target`. Hoy no lo cambiamos.
