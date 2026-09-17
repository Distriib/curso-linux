# 3 — Cómo arranca RHEL, GRUB, targets y `rd.break` (guía del instructor)

Diez minutos. Tienen que quedar cuatro cosas: (1) los cinco pasos del arranque y **en qué paso** se atasca cada síntoma; (2) en GRUB hay un archivo que se edita (`/etc/default/grub`) y uno que se genera (`grub.cfg`), y los kernels se manejan con `grubby`; (3) la receta de `rd.break` y **por qué** el `/.autorelabel`; (4) la receta para salir de emergency mode. Tipeás los cinco comandos del bloque bash mientras explicás la tabla de archivos; el resto se lee por encima y se practica en los tres labs.

**Qué es el firmware (BIOS / UEFI):** el programa que viene en la placa y arranca antes que todo. BIOS es el esquema viejo (busca el cargador en el primer sector del disco); UEFI es el moderno (lo busca en una partición `/boot/efi`). VirtualBox usa BIOS salvo que se active EFI; tu UTM usa UEFI siempre.
**Qué es un cargador de arranque:** el programa chico que el firmware ejecuta y cuyo único trabajo es cargar el kernel. En RHEL es GRUB2.
**Qué es GRUB2:** *GRand Unified Bootloader*. Muestra el menú de kernels, carga el que se elija junto con su initramfs, y le pasa los parámetros de la línea `linux`.
**Qué es el kernel:** el núcleo del sistema: maneja hardware, memoria y procesos. Es el archivo `/boot/vmlinuz-VERSION`.
**Qué es el initramfs:** un sistema mínimo comprimido (`/boot/initramfs-VERSION.img`) que el kernel carga en memoria para tener los drivers y LVM necesarios para **encontrar y montar** el disco real. Cuando lo logra, monta la raíz real en `/sysroot` y le pasa el control (a eso se le dice "pivotar").
**Qué es `/sysroot`:** dentro del initramfs, el punto donde está montado el disco real. Todo lo que uno quiera tocar del sistema de verdad está ahí.
**Qué es PID 1:** el primer proceso, el padre de todos: systemd. Si muere, el kernel se detiene (*kernel panic*).
**Qué es un target:** una unidad de systemd que agrupa otras y define "hasta dónde arrancar" (Día 4). `default.target` es un enlace al que se usa al encender.
**Qué es `multi-user.target` / `graphical.target`:** servidor completo sin escritorio / con escritorio. El de esta VM es `multi-user`.
**Qué es `rescue.target`:** el antiguo "modo monousuario": monta todo lo de `fstab`, arranca lo básico, sin red, y pide la contraseña de root. Para arreglar algo que impide el arranque normal.
**Qué es `emergency.target`:** menos todavía: solo la raíz, en solo lectura, y la contraseña de root. Systemd cae ahí **solo** cuando falla un montaje obligatorio de `fstab`. Es el Lab 3.3.
**Qué es `grub rescue>`:** el prompt que muestra GRUB cuando ni siquiera encuentra sus propios archivos. Disco cambiado, partición borrada, GRUB roto.
**Qué es un kernel panic:** el kernel se detiene porque no puede seguir. `Unable to mount root fs` = no encontró la raíz: initramfs sin el driver, o el `root=` de la línea `linux` apunta mal. Se arregla eligiendo otro kernel en el menú.
**Qué es `/etc/default/grub`:** el archivo de configuración que sí se edita. `GRUB_TIMEOUT` = segundos que espera el menú; `GRUB_CMDLINE_LINUX` = parámetros que se le pasan al kernel; `GRUB_DEFAULT=saved` = arranca el último elegido.
**Qué es `rhgb quiet`:** dos parámetros del kernel: `rhgb` = pantalla gráfica de arranque, `quiet` = no mostrar mensajes. Se quitan para ver qué hace el sistema al arrancar.
**Qué es `grub2-mkconfig`:** el comando que **genera** `/boot/grub2/grub.cfg` a partir de `/etc/default/grub`. `grub.cfg` no se edita a mano: se regenera. En RHEL 9 el destino es ese aunque la VM sea UEFI.
**Qué es BLS / `/boot/loader/entries/`:** *Boot Loader Specification*: cada kernel instalado tiene su propio archivo `.conf` con su línea de arranque. GRUB los lee de ahí; `grub2-mkconfig` no los toca.
**Qué es `grubby`:** la herramienta para leer y cambiar esas entradas: qué kernel es el default, qué parámetros lleva cada uno. Es lo que pide el examen.
**Qué es `grubenv` / `grub2-editenv list`:** un archivo chico con variables que GRUB guarda entre arranques (`saved_entry` = último kernel elegido, `boot_success`, `menu_auto_hide` = ocultar el menú si el último arranque salió bien). `grub2-editenv list` las muestra; `grub2-editenv - unset VARIABLE` borra una.
**Qué es `menu_auto_hide`:** la variable que hace que GRUB **no muestre** el menú si el arranque anterior fue bien. Por eso los estudiantes "nunca vieron GRUB". Se quita en el Lab 3.1.
**Qué es la entrada `rescue`:** una entrada extra del menú con un initramfs genérico que trae **todos** los drivers. Para cuando el normal no arranca.
**Qué es `rpm -q kernel`:** lista los kernels instalados. `dnf` conserva hasta 3 y nunca borra el que está en uso: siempre hay uno de respaldo.
**Qué es `rd.break`:** un parámetro del kernel que le dice al initramfs "detenete antes de pasar el control al sistema real". Abre una shell con el disco en `/sysroot`. Sin contraseña: por eso el acceso a la consola equivale a root.
**Qué es `e` y `Ctrl+X` en GRUB:** `e` abre el editor de la entrada seleccionada (se pueden cambiar los parámetros para **este** arranque, no queda guardado); `Ctrl+X` arranca con lo editado.
**Qué es `mount -o remount,rw`:** volver a montar un disco que estaba en solo lectura (`ro`) para poder escribir (`rw`). Hace falta en `/sysroot` con `rd.break` y en `/` en emergency mode.
**Qué es `chroot /sysroot`:** "cambiar la raíz": a partir de ahí, `/` pasa a ser `/sysroot`. Así `passwd` escribe en el `/etc/shadow` **del disco**, no en el del initramfs.
**Qué es `/.autorelabel`:** un archivo vacío en la raíz. Si existe al arrancar, SELinux vuelve a poner la etiqueta correcta a **todos** los archivos del disco y reinicia. Tarda 1 a 5 minutos (más en UTM emulado).
**Por qué hace falta el relabel:** en el initramfs no hay política SELinux cargada. `passwd` reescribe `/etc/shadow`, y el archivo nuevo queda **sin etiqueta** (`unlabeled_t`). Con la política cargada, nadie tiene permiso para leer un archivo sin etiqueta: ningún login funcionaría. Es un segundo ticket escondido dentro del primero.
**Qué es `shadow_t` / `unlabeled_t`:** la etiqueta SELinux correcta de `/etc/shadow` / la de un archivo creado sin política. `ls -Z` la muestra (Día 8).
**Qué es `systemd.unit=rescue.target`:** parámetro del kernel para arrancar **una vez** en ese target, sin cambiar el default. Se escribe en GRUB con `e`, igual que `rd.break`.
**Qué es `journalctl -xb`:** el log de **este** arranque (`-b`) con explicaciones (`-x`): systemd agrega debajo de cada error qué significa y qué mirar. Es lo que sugiere el propio emergency mode.
**Qué es `systemctl daemon-reload`:** hacer que systemd relea la configuración, incluido `/etc/fstab` (systemd genera una unidad `.mount` por cada línea). Después de editar `fstab`, siempre.
**Qué es `mount -a`:** montar todo lo de `fstab` que no esté montado. Si una línea está mal, acá falla, **antes** de reiniciar (Día 6).
**Qué es `findmnt --verify`:** revisar `fstab` línea por línea y avisar de las que no van a montar (Día 6). En la versión de RHEL 9, cuando todo está bien dice `Success, no errors or warnings detected`; si hay problemas, `0 parse errors, 0 errors, 1 warning`.
**Qué es `systemctl default`:** desde emergency o rescue, seguir arrancando hasta el target por defecto sin reiniciar.
**Qué es `nofail`:** opción de `fstab` que dice "si este disco no está, seguí arrancando igual" (Día 6). Sin ella, un disco de datos ausente tumba el arranque entero. Va en discos de datos; nunca en `/`, `/boot` ni `/var`.
**Qué es un UUID:** el identificador único de un sistema de archivos (Día 6). `deadbeef-...` es un UUID inventado: ningún disco lo tiene, y por eso el montaje falla.
**Qué es `grub2-setpassword`:** ponerle contraseña a GRUB para que `e` pida credenciales. Obligatorio en producción; en el laboratorio no, porque impediría el Lab 3.2.
**Qué es la consola de la VM:** la ventana de VirtualBox o UTM, equivalente a estar sentado frente al servidor con teclado y monitor. Ahí se ve GRUB y el emergency mode; por SSH no.

---

## El arranque en cinco pasos

**Qué decir:** "saber en qué paso se detuvo es la mitad del diagnóstico. Si ven `grub rescue>` es el paso 2; si ven `Kernel panic` es el 3; si ven `emergency mode` es el 4: casi siempre `fstab`."
**Qué señalar:** la fila del paso 4 es el Lab 3.3; la del 5 es lo que ya hicieron en el Lab 1.1.

---

## Archivos de GRUB2

Tipear los cinco comandos. **Qué señalar:** `GRUB_TIMEOUT=5` (en el lab pasa a 10), `rhgb quiet` al final de `GRUB_CMDLINE_LINUX` (en el lab se quitan), y la carpeta `entries/` con dos archivos: el kernel y el `rescue`. **La frase:** "`/etc/default/grub` se edita; `grub.cfg` se genera. Si editan `grub.cfg` a mano, el próximo `grub2-mkconfig` lo pisa."

---

## `grubby`

Señalar solo `--remove-args`: "es lo que vamos a usar para quitar `rhgb quiet` de todos los kernels de una vez, y queda guardado."

---

## Targets

**Qué decir:** "`rescue` es el modo monousuario de antes: para arreglar un servicio. `emergency` es donde systemd cae **solo**, cuando un montaje de `fstab` falla: `/` en solo lectura y la contraseña de root." Los dos piden la contraseña de root: por eso primero se recupera la contraseña (Lab 3.2) y después se practica emergency (Lab 3.3).

---

## Contraseña de root perdida: `rd.break`

Leer el diagrama de arriba a abajo, despacio. **La frase que hay que decir completa:** "en el initramfs SELinux no está cargado. `passwd` escribe un `/etc/shadow` nuevo **sin etiqueta**. Si arrancan así, con SELinux ya cargado, nadie puede leer ese archivo y **nadie** puede iniciar sesión, ni root. `touch /.autorelabel` hace que el próximo arranque etiquete todo de nuevo. Sin esa línea, arreglaron un problema y crearon otro peor."

---

## Salir de emergency mode

Leer la receta. **Qué señalar:** el orden `daemon-reload` → `mount -a` → `findmnt --verify` → `systemctl default`: "si `mount -a` sigue quejándose, no salgan: van a caer otra vez en emergency." Cerrar con la idea del snapshot: "en una VM, antes de tocar GRUB o `fstab`, snapshot. Es la recuperación más rápida que existe."
