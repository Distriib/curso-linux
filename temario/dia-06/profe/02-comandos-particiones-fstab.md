# Comandos — particionar, formatear, montar, `fstab` y swap (guía del instructor)

Diez minutos. Se lee de arriba a abajo señalando las tablas; no se particiona ni se formatea acá: todo eso se hace en los cuatro labs que siguen. Lo único que podés correr para mostrar: `cat /etc/fstab`, `swapon --show`, `free -m`. Tres cosas que tienen que quedar: (1) `parted` escribe al instante y `fdisk` solo con `w`; (2) después de tocar `fstab`: `daemon-reload`, `mount -a`, `findmnt --verify`, y `nofail` en los discos de datos; (3) la swap es disco que hace de RAM de emergencia.

**Qué es `parted`:** el programa para crear y borrar particiones. Se abre sobre un disco (`sudo parted /dev/sdb`) y adentro se dan órdenes. Cada orden se escribe en el disco **al momento**: no hay "deshacer" ni "guardar".
**Qué es `mklabel gpt`:** crear la tabla de particiones en formato GPT. "Label" acá no es la etiqueta del sistema de archivos: es el nombre viejo de la tabla de particiones (*disk label*). Borra cualquier tabla anterior sin preguntar.
**Qué es `unit MiB`:** decirle a `parted` que muestre e interprete todos los números en MiB. Sin esto, mezcla MB, GB y sectores y los tamaños no cuadran.
**Qué es `mkpart`:** crear una partición: nombre, tipo pensado, inicio, fin. `primary` en GPT es solo el nombre de la partición (en MBR era un tipo); se escribe por costumbre. `xfs` o `linux-swap` es una nota de intención: `parted` no crea el sistema de archivos, eso lo hace `mkfs`.
**Qué es `1MiB` como inicio:** el primer MiB del disco es de la tabla GPT. Empezar en 0 rompe la tabla; empezar en 1MiB deja todo alineado.
**Qué es `100%`:** "hasta el final del disco". Evita calcular el último MiB.
**Qué es `set 3 lvm on`:** marcar la partición 3 con el tipo "Linux LVM". No es obligatorio para que LVM funcione; documenta el propósito para quien mire el disco después.
**Qué es `rm 3`:** borrar la partición 3 (solo la entrada en la tabla; los datos siguen ahí hasta que algo los pise).
**Qué es `fdisk`:** el otro programa para particionar, más viejo, con teclas de una letra. Acumula los cambios en memoria y los escribe solo con `w`; `q` sale sin tocar nada. Por eso es más seguro para practicar.
**Qué es `+3G` en `fdisk`:** el fin de la partición dado como tamaño: "3 GiB desde el inicio". Con `Enter` se toma el valor propuesto (el primer sector libre; el fin del disco).
**Qué es `udev` y `udevadm settle`:** `udev` es el servicio que crea los archivos de `/dev` cuando aparece hardware o una partición. `udevadm settle` espera a que termine, para que `lsblk` ya muestre `sdb1`, `sdb2`...
**Qué es `partprobe`:** pedirle al kernel que relea la tabla de particiones de un disco. Solo hace falta si `lsblk` no muestra las particiones nuevas.
**Qué es `mkfs`:** *make filesystem*: formatear. `mkfs.xfs`, `mkfs.ext4`, `mkfs.vfat` son un programa por tipo. Tarda un segundo porque escribe solo la estructura (los metadatos), no borra byte a byte. Todo lo que había queda inaccesible.
**Qué es `-f` de `mkfs.xfs`:** *force*: `mkfs.xfs` se niega a pisar una partición que ya tiene una firma; `-f` lo obliga. `mkfs.ext4`, en cambio, pregunta `Proceed anyway?`.
**Qué es `-L` de `mkfs.xfs`:** poner la etiqueta al crear. Equivale a `xfs_admin -L` después.
**Qué es `xfs_admin -L`:** cambiar la etiqueta de un XFS ya creado. Exige que esté desmontado; máximo 12 caracteres.
**Qué es `e2label`:** lo mismo para ext4 (la `e2` viene de ext2, el abuelo de ext4).
**Qué es `dosfstools`:** el paquete que trae `mkfs.vfat`. No viene instalado.
**Qué es `wipefs -a`:** borrar todas las firmas de una partición (la del sistema de archivos, la de LVM): la deja como recién creada. Se usa antes de entregarle un disco usado a Stratis.
**Qué es `blkid -s UUID -o value`:** `-s UUID` = mostrar solo ese campo; `-o value` = solo el valor, sin `UUID=` ni comillas. Da el UUID pelado para copiarlo a `fstab`.
**Qué es `mount`:** enganchar un sistema de archivos en una carpeta. `mount dispositivo carpeta`. Dura hasta el próximo reinicio.
**Qué es `-o`:** *options*: opciones de montaje separadas por coma. `ro` = solo lectura; `rw` = lectura y escritura; `remount` = cambiar opciones de algo ya montado, sin desmontar.
**Qué es `umount`:** desenganchar. Se escribe sin la `n`: `umount`, no `unmount`. Es el error de tipeo del día.
**Qué es `findmnt`:** mostrar qué está montado, de qué dispositivo, de qué tipo y con qué opciones. `findmnt /archivos` para uno; solo `findmnt` para el árbol completo.
**Qué es `target is busy`:** el mensaje de `umount` cuando algún proceso tiene un archivo abierto o está parado (con `cd`) dentro de la carpeta. No se puede desmontar hasta que se vaya.
**Qué es `fuser -vm`:** *file user*: qué procesos usan un sistema de archivos (`-m`), en forma legible (`-v`). La columna `ACCESS` dice cómo: `c` = tiene esa carpeta como directorio actual, `f` = archivo abierto. Viene en el paquete `psmisc`, que está instalado.
**Qué es `lsof`:** *list open files*: parecido a `fuser`, más detallado. Puede no estar instalado; por eso el material usa `fuser`.
**Qué es `/etc/fstab`:** *filesystem table*: el archivo que lee el sistema al arrancar para saber qué montar y dónde. Seis campos por línea, separados por espacios o tabulaciones.
**Qué es `defaults`:** el conjunto de opciones estándar: lectura y escritura, montar al arrancar, y varias más que no importan hoy.
**Qué es `nofail`:** "si este dispositivo no está, seguí arrancando igual". Sin `nofail`, systemd espera 90 segundos al dispositivo y después cae en modo de emergencia. Va en todo disco de datos; no va en `/` ni en `/boot`.
**Qué es `noauto`:** no montar al arrancar (se monta a mano o bajo demanda).
**Qué es `dump` (campo 5):** un programa de respaldo de los años 80 que ya nadie usa; el campo queda en `0`.
**Qué es `fsck` y el campo 6 (`pass`):** `fsck` revisa un sistema de archivos al arrancar. El campo dice en qué orden: `1` para `/`, `2` para otros ext4, `0` = no revisar. XFS y swap llevan `0` (XFS se revisa solo, de otra forma).
**Qué es `daemon-reload`:** ya lo vieron el Día 4 con los servicios: systemd relee su configuración. Acá hace falta porque systemd convierte cada línea de `fstab` en una unidad `.mount` y tiene que regenerarlas. Si se olvida, RHEL 9 avisa al hacer `mount`: `(hint) your fstab has been modified, but systemd still uses the old version`.
**Qué es `mount -a`:** montar todo lo de `fstab` que todavía no está montado. Es la prueba antes de reiniciar: un error en la línea sale ahora, en pantalla, y no en el arranque.
**Qué es `findmnt --verify`:** revisar `fstab` línea por línea: sintaxis, que el UUID exista, que la carpeta exista. Dice `Success, no errors or warnings detected` si está todo bien.
**Qué es el modo de emergencia:** la pantalla en la que cae la VM cuando algo de `fstab` falla al arrancar y no tenía `nofail`. Pide la contraseña de root, no hay red ni SSH: hay que ir a la ventana de la VM. Se sale corrigiendo `fstab` y con `systemctl default` (seguir el arranque normal). El `mount -o remount,rw /` es porque en ese modo `/` puede estar en solo lectura y `vi` no podría guardar.
**Qué es swap:** espacio en disco que el kernel usa como extensión de la RAM cuando se agota. No hace más rápido al servidor: evita que se caiga. La VM trae 2 GiB en `rhel-swap`.
**Qué es `mkswap`:** el `mkfs` de la swap: prepara la partición (o el archivo) y le da un UUID.
**Qué es `swapon` / `swapoff`:** activar / desactivar una swap. `swapon --show` lista las activas; `swapon -a` activa todas las de `fstab`.
**Qué es `PRIO`:** prioridad: el kernel usa primero la swap de número más alto. Las negativas las asigna solo, en orden de activación (`-2`, `-3`). Sirve para preferir una swap que esté en un disco más rápido.
**Qué es `free -m`:** memoria RAM y swap, total, ocupada y libre, en MB. La línea `Swap:` es la que importa hoy.
**Qué es `dm-1`:** el nombre interno del segundo dispositivo de device mapper: `rhel-swap`. `swapon --show` muestra ese nombre en vez de `/dev/mapper/rhel-swap`.
**Qué es `dd`:** copia bytes de un lado a otro, en bloques. `if=` de dónde lee, `of=` a dónde escribe, `bs=` tamaño de bloque, `count=` cuántos bloques. `status=progress` muestra el avance. Es peligroso si se equivoca el `of=`: puede pisar un disco entero.
**Qué es `/dev/zero`:** un dispositivo falso que da ceros sin fin. `dd if=/dev/zero of=/swapfile bs=1M count=512` escribe 512 MiB de ceros en el archivo: así el archivo ocupa de verdad todo su tamaño.
**Qué es `fallocate`:** crea un archivo grande al instante, sin escribirlo. Sobre XFS deja "huecos" y `swapon` rechaza el archivo. Por eso para swap siempre `dd`.
**Qué es `chmod 600 /swapfile`:** solo root lee y escribe (Día 3). `mkswap` avisa si el archivo lo puede leer cualquiera: la swap tiene pedazos de la memoria de todos los procesos adentro.
**Qué es SELinux y `restorecon`:** SELinux es la capa de seguridad de RHEL que le pone a cada archivo una etiqueta de contexto y controla qué programa puede tocar qué (es el `context=` que vieron en `id` y el `seclabel` de `findmnt`; es el Día 8 completo). `restorecon -v archivo` le pone al archivo la etiqueta que la política dice que debe tener y, con `-v`, muestra si la cambió. Para `/swapfile` es la práctica correcta; si no imprime nada, ya estaba bien.

---

## `parted` y `fdisk`

No correrlos acá. **Qué decir:** *"dos herramientas, mismo resultado. `parted` escribe al instante: lo que tipean ya está en el disco. `fdisk` acumula y escribe solo con `w`; con `q` se van sin tocar nada. Vamos a usar las dos, una en cada disco."*
**Qué señalar:** en el diagrama de `mkpart`, el `1MiB` de inicio (*"nunca 0"*) y la palabra `xfs` (*"es una nota, no formatea"*). En la tabla de `fdisk`, `q` y `w`.

**La frase del día:** *"antes de abrir `parted` o `fdisk`, `lsblk`. Si abren `parted /dev/sda` y hacen `mklabel gpt`, borran la tabla del disco del sistema, sin preguntar. La única salida es restaurar el snapshot."*

---

## Formatear y etiquetar

**Qué señalar:** las dos filas de `mkfs.xfs`: sin `-f` se niega si hay algo; y `-L` pone la etiqueta de una. La fila de `wipefs`: *"la vamos a necesitar si alguien le da a Stratis una partición usada"*.
**Qué decir:** *"`mkfs` es un segundo, pero es destructivo: lo que había ya no se ve, y el UUID es otro. Si `fstab` tenía el UUID viejo, el próximo arranque falla."*

---

## Montar y desmontar

**Qué señalar:** `umount` sin `n`. `findmnt` como "¿está montado?". `fuser -vm` para el `target is busy` del Lab 2.3.

---

## `/etc/fstab`

`cat /etc/fstab` en tu VM (ya lo leyeron en el Lab 1.1). Leer el diagrama de los seis campos de izquierda a derecha, sobre la línea de `/boot`.
**Qué decir, despacio:** *"campo 1, qué: siempre UUID, etiqueta o nombre de LVM, nunca `sdb1`, porque mañana puede ser `sdc1`. Campo 2, dónde: la carpeta tiene que existir. Campo 3, tipo. Campo 4, opciones: `defaults` y, en discos de datos, `nofail`. Campos 5 y 6: cero y cero."*
**Qué señalar:** el comentario que trae el propio archivo: *"Red Hat mismo escribió ahí: después de editar, `systemctl daemon-reload`."*

**La frase que hay que decir completa:** *"después de tocar `fstab`, siempre los tres: `daemon-reload`, `mount -a`, `findmnt --verify`. Si `mount -a` da error, lo ven ahora, en su terminal. Si reinician sin probar, el error lo ven en una pantalla negra que pide la clave de root, sin red, sin SSH. Eso es el modo de emergencia."*

De la caja de emergencia: leerla una vez, tranquilo. *"Si a alguien le pasa hoy, lo resolvemos juntos con estos cinco pasos. No es el fin del mundo: es un `#` en una línea."*

---

## Swap

`swapon --show` y `free -m` en tu VM: una swap de 2G.
**Qué decir:** *"la swap es disco que hace de RAM cuando la RAM se acaba. No acelera nada: evita que el servidor se caiga. Regla práctica: con menos de 2 GB de RAM, el doble; entre 2 y 8, lo mismo que la RAM; con más, 4 a 8 GB."*
**Qué señalar:** la línea de `fstab` para swap tiene `swap` en el campo 2 y en el 3 (el instalador puso `none` en el 2: vale igual). Del archivo swap: *"esto lo hacen en el reto. `dd`, nunca `fallocate`; `chmod 600` obligatorio."* Leer el diagrama de `dd` una vez.
