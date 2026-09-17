# Lab — Contraseña de root perdida (comandos)

Guiado, todos en la ventana de su VM. Vos compartís la ventana de la tuya. Antes de empezar: que todos tengan a mano la contraseña de root actual (se pone **la misma**). En tu UTM el menú puede no aparecer solo: pulsar `Esc` apenas arranca.

## Parte 1
```bash
sudo systemctl reboot
```
En el menú de GRUB: una flecha (congela la cuenta), después `e`.

## Parte 2
Flecha abajo hasta la línea `linux`, `Ctrl+E` (o `Fin`) para ir al final, espacio, y escribir:
```
rd.break
```
`Ctrl+X`.
Si en vez del prompt `switch_root:/#` pide contraseña de root (pasa en algunas 9.x), usar la alternativa del final de este archivo.

## Parte 3
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
```
Si `passwd` dice `Authentication token manipulation error`: faltó el `remount,rw` o el `chroot`. Repetir los dos y volver a `passwd`.

## Parte 4
```bash
touch /.autorelabel
exit
exit
```
Arranca, reetiqueta (fila de `*`), reinicia solo. En tu UTM tarda más que en sus VirtualBox: avisarlo. **Descanso de 15 minutos acá.** Si alguien apaga la VM a mitad del relabel, al encender lo repite: no pasa nada.

## Parte 5 — al volver
```bash
su -
ls -Z /etc/shadow
journalctl -b -1 -n 3 --no-pager
exit
```
Si a alguien `ls -Z` le da `unlabeled_t` o no puede entrar: `sudo touch /.autorelabel` y `sudo reboot` (si tampoco puede con `sudo`, volver a hacer `rd.break` y dentro del `chroot` el `touch /.autorelabel`).

---

## Alternativa si `rd.break` pide contraseña

En GRUB, `e`, al final de la línea `linux` agregar `init=/bin/bash`, `Ctrl+X`. Aparece `bash-5.1#` con el disco real en `/`, solo lectura:
```bash
mount -o remount,rw /
passwd root
touch /.autorelabel
exec /usr/lib/systemd/systemd
```
Sin `chroot`, porque acá bash ya corre sobre el sistema real. Si el `exec` no continúa el arranque: `/usr/sbin/reboot -f`.
