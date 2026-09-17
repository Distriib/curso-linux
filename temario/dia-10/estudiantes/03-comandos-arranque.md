# 3 — Cómo arranca RHEL, GRUB, targets y `rd.break`

## El arranque en cinco pasos

Saber en qué paso se detuvo el servidor es la mitad del diagnóstico.

```
1 Firmware (BIOS o UEFI)   busca el cargador de arranque en el disco
2 GRUB2                    muestra el menú; carga el kernel y el initramfs
3 Kernel + initramfs       el initramfs trae drivers y LVM para encontrar /, lo monta y le pasa el control
4 systemd (PID 1)          lee /etc/fstab, monta todo, arranca los servicios hasta el target por defecto
5 default.target           multi-user.target: consola y sshd listos
```

| Se atasca en | Se ve en pantalla | Causa típica | Hoy |
|---|---|---|---|
| 1–2 | `No bootable device` · `grub rescue>` | disco u orden de arranque; GRUB dañado | — |
| 3 | `Kernel panic ... Unable to mount root fs` | initramfs sin el driver; elegir **otro kernel** en GRUB | — |
| 4 | `You are in emergency mode` · `Give root password for maintenance` | una línea de `/etc/fstab` que no monta | Lab 3.3 |
| 5 | arranca, pero un servicio falla | `systemctl --failed` | Lab 1.1 |

## Archivos de GRUB2

| Archivo | Qué es | Se toca |
|---|---|---|
| `/etc/default/grub` | lo que uno edita: `GRUB_TIMEOUT`, `GRUB_CMDLINE_LINUX` | con `vim`; después `grub2-mkconfig` |
| `/boot/grub2/grub.cfg` | el menú **generado** | no: `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` lo regenera |
| `/boot/loader/entries/*.conf` | una entrada por kernel | con `grubby` |
| `/boot/grub2/grubenv` | variables guardadas (`saved_entry`, `menu_auto_hide`) | `sudo grub2-editenv list` |

```bash
cat /etc/default/grub
sudo grubby --default-kernel
sudo grubby --info=DEFAULT
ls /boot/loader/entries/
rpm -q kernel
```

## `grubby`

| Comando | Qué hace |
|---|---|
| `sudo grubby --default-kernel` | qué kernel arranca |
| `sudo grubby --info=ALL` | todas las entradas con sus parámetros |
| `sudo grubby --set-default /boot/vmlinuz-VERSION` | elegir otro kernel por defecto |
| `sudo grubby --update-kernel=ALL --remove-args="rhgb quiet"` | quitar parámetros a **todos** los kernels, persistente |
| `sudo grubby --update-kernel=ALL --args="algo"` | agregar parámetros |

`dnf install kernel` siempre conserva el anterior: si el nuevo no arranca, en GRUB se elige el viejo.

## Targets

| Target | Qué arranca | Para qué |
|---|---|---|
| `multi-user.target` | todo, sin gráfico | el normal de un servidor |
| `graphical.target` | todo, más escritorio | estaciones de trabajo |
| `rescue.target` | lo básico; monta `fstab`; sin red; pide clave de root | arreglar un servicio que impide arrancar |
| `emergency.target` | lo mínimo; `/` en solo lectura; pide clave de root | donde cae systemd **solo** cuando falla un montaje de `fstab` |

| Comando | Qué hace |
|---|---|
| `systemctl get-default` | cuál es el target por defecto |
| `sudo systemctl set-default multi-user.target` | cambiarlo |
| en GRUB: `e`, al final de la línea `linux`: `systemd.unit=rescue.target`, `Ctrl+X` | arrancar **una vez** en rescate |

## Contraseña de root perdida: `rd.break`

Detener el arranque **dentro del initramfs**: el disco ya se ve en `/sysroot`, pero el sistema real todavía no tomó el control.

```
GRUB  →  e  →  final de la línea linux  →  rd.break  →  Ctrl+X

switch_root:/#  mount -o remount,rw /sysroot
switch_root:/#  chroot /sysroot
sh-5.1#         passwd root
sh-5.1#         touch /.autorelabel
sh-5.1#         exit
switch_root:/#  exit
```

`/.autorelabel`: en el initramfs SELinux no está cargado, así que el `/etc/shadow` nuevo nace **sin etiqueta** y nadie podría iniciar sesión. Ese archivo hace que el próximo arranque reetiquete todo el disco (1 a 5 minutos) y reinicie solo.

## Salir de emergency mode

```
Give root password for maintenance   →   contraseña de root

journalctl -xb -p err --no-pager | grep -i depend
systemctl --failed
mount -o remount,rw /
vi /etc/fstab                (borrar o comentar la línea mala)
systemctl daemon-reload
mount -a
findmnt --verify
systemctl default
```

El acceso a la consola equivale a root. Por eso en producción GRUB lleva contraseña (`grub2-setpassword`) y el centro de datos tiene puerta con llave. En una VM, la red de seguridad es el **snapshot**: antes de tocar GRUB o `fstab`, snapshot.
