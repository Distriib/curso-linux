# Lab — GRUB visible y kernels (comandos)

Guiado: todos tipean, se lee la salida juntos. Este lab es obligatorio para los dos siguientes: sin menú visible no hay `rd.break`.

## Parte 1
```bash
cat /etc/default/grub
sudo grub2-editenv list
sudo grubby --default-kernel
sudo grubby --info=DEFAULT
ls /boot/loader/entries/
rpm -q kernel
```
En tu VM (UTM, aarch64) el kernel termina en `.aarch64` y `crashkernel=` puede tener otro valor: decirlo.

## Parte 2
```bash
sudo grub2-editenv - unset menu_auto_hide
sudo vim /etc/default/grub
```
En vim: `/GRUB_TIMEOUT` `Enter`, cambiar el `5` por `10`, `Esc`, `:wq`.
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
grep GRUB_TIMEOUT /etc/default/grub
sudo grub2-editenv list
```
Si `grub2-editenv - unset menu_auto_hide` no imprime nada, está bien: no imprime nada nunca.

## Parte 3
```bash
sudo grubby --update-kernel=ALL --remove-args="rhgb quiet"
sudo grubby --info=DEFAULT | grep args
```
Decir: "esto cambia las entradas de los kernels **ya instalados**; `GRUB_CMDLINE_LINUX` en `/etc/default/grub` solo afecta a los kernels que se instalen después. Por eso las dos cosas."

## Parte 4
```bash
systemctl get-default
systemctl list-dependencies rescue.target --no-pager | head -6
systemctl cat emergency.target | grep Requires
```
