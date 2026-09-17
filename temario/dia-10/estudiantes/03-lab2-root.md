# Lab 3.2 — Contraseña de root perdida

Objetivo: recuperar el acceso de root en una VM cuya contraseña "se perdió", con el procedimiento oficial de RHEL 9. Se trabaja **en la ventana de la VM** (VirtualBox), no por SSH. La contraseña nueva será **la misma que ya tenía root**, para no desincronizar al grupo.

---

## Parte 1 — Detener el arranque en GRUB

**¿Dónde se interrumpe el arranque para entrar sin contraseña?**

```bash
sudo systemctl reboot
```
En la ventana de la VM, cuando aparece el menú de GRUB (dura 10 s), pulsar **una flecha** para congelar la cuenta, y después **`e`** sobre la primera entrada.

**Comprobar:** se ve el editor de GRUB con varias líneas; una empieza con `linux` (o `linuxefi`) y termina con `rd.lvm.lv=rhel/swap`, ya sin `rhgb quiet`.

---

## Parte 2 — Pedirle al initramfs que se detenga

**¿Qué le agrego al kernel para que pare antes de arrancar el sistema?**

Bajar hasta la línea `linux`, ir al **final** de la línea (`Ctrl+E` o tecla `Fin`), agregar un espacio y:
```
rd.break
```
Pulsar **`Ctrl+X`** para arrancar con esa línea. No queda guardada: es solo para este arranque.

**Comprobar:** pasan los mensajes del arranque y se detiene en:
```
Press Enter for maintenance
(or press Control-D to continue):
switch_root:/#
```
(pulsar `Enter` si lo pide). Acá `/` es el initramfs, y el disco real está en `/sysroot`, en **solo lectura**.

---

## Parte 3 — Cambiar la contraseña

**¿Cómo escribo en el disco real desde acá?**

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
```
Escribir la contraseña de root **dos veces** (la misma de siempre).

**Comprobar:**
```
Changing password for user root.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```
Si avisa `BAD PASSWORD`, root puede insistir: la acepta igual.

---

## Parte 4 — Pedir el reetiquetado y seguir

**¿Por qué no basta con cambiar la contraseña?**

```bash
touch /.autorelabel
exit
exit
```

**Comprobar:** el sistema sigue arrancando, muestra `*** Warning -- SELinux targeted policy relabel is required` con una fila de `*` que avanza, y **reinicia solo** al terminar (1 a 5 minutos). No tocar nada. **Acá empieza el descanso.**

---

## Parte 5 — Al volver: comprobar

**¿Funciona la contraseña, y quedó bien `/etc/shadow`?**

Por SSH, como siempre:
```bash
su -
ls -Z /etc/shadow
journalctl -b -1 -n 3 --no-pager
exit
```

**Comprobar:**
```
Password:
[root@rhel01 ~]#
system_u:object_r:shadow_t:s0 /etc/shadow
... Reached target ... Reboot ...
```
`shadow_t` es el contexto correcto: el reetiquetado hizo su trabajo. Si dijera `unlabeled_t`, faltó el `/.autorelabel` y nadie podría entrar.
