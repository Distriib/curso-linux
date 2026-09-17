# Comandos — NFS y autofs (guía del instructor)

Ocho minutos, sin tipear: es lectura de diagramas. Tres cosas que tienen que quedar: (1) el servidor **exporta** con `/etc/exports` y `exportfs -rav`; el cliente **monta** con `mount -t nfs`; (2) la línea de `fstab` lleva `_netdev,nofail`; (3) autofs monta cuando alguien entra a la carpeta y la suelta solo, y es lo que pide el examen RHCSA. Hoy la misma VM es servidor y cliente.

**Qué es NFS:** *Network File System*. El protocolo clásico de Linux y Unix para usar una carpeta de otro servidor como si fuera local. Windows usa SMB (Bloque 3); Linux entre Linux usa NFS.
**Qué es exportar:** publicar una carpeta por NFS. El servidor dice "esta carpeta, para estas máquinas, con estas opciones". Se escribe en `/etc/exports` y se aplica con `exportfs`.
**Qué es montar:** lo mismo que el Día 6 con un disco: colgar algo en una carpeta vacía. Acá lo que se cuelga viene por la red.
**Qué es NFSv4.2:** la versión que usa RHEL 9 por defecto. Un solo puerto, `2049/tcp`. Las versiones viejas (v3) necesitan además `rpcbind` (111) y `mountd` (20048).
**Qué es `nfs-utils`:** el paquete con todo NFS, servidor y cliente. Probablemente ya está instalado.
**Qué es `nfs-server`:** el servicio del servidor. Queda `active (exited)`: el trabajo lo hace el kernel, no un proceso que se vea en `ps`. Es normal.
**Qué es `/etc/exports`:** la lista de exportaciones. Una línea por carpeta: ruta, espacio, cliente y opciones **pegados** entre paréntesis. `192.168.56.0/24` = toda la red host-only; `127.0.0.1` = solo esta máquina.
**Qué es `rw` / `ro`:** lectura y escritura / solo lectura. Lo decide el servidor: un cliente puede montar `rw` un export `ro` y va a chocar igual.
**Qué es `sync`:** el servidor confirma cada escritura cuando llegó al disco. Más seguro, un poco más lento. Se pone siempre.
**Qué es `root_squash`:** viene por defecto: si el root del cliente escribe, en el servidor aparece como `nobody` (Día 3: la cuenta sin permisos). Protege al servidor de los root de las otras máquinas. `no_root_squash` lo apaga: útil en laboratorio, peligroso en producción.
**Qué es `exportfs -rav`:** aplicar el archivo. `-r` releer todo, `-a` todas las entradas, `-v` contar qué hizo. `exportfs -v` solo muestra lo que está exportado, con todas las opciones, incluidas las que no escribimos (`wdelay`, `no_subtree_check`, `sec=sys`).
**Qué es `sec=sys`:** NFS confía en el UID que dice el cliente. El UID 1000 del cliente es el UID 1000 del servidor, sin contraseña. Alcanza en una red controlada; para más, Kerberos.
**Qué es `showmount -e`:** preguntarle a un servidor qué exporta. Necesita `rpc-bind` y `mountd` abiertos en el firewall del servidor.
**Qué son `nfs`, `rpc-bind`, `mountd` en el firewall:** los tres servicios que hay que abrir. Con `nfs` solo alcanza para clientes v4; los otros dos son para `showmount` y clientes viejos. Se abren los tres, en las dos zonas.
**Qué es `mount -t nfs servidor:/ruta /punto`:** montar a mano. `-t nfs` = el tipo; `servidor:/ruta` = de dónde; `/punto` = dónde. Se ve con `mount | grep nfs4` y con `df -hT`.
**Qué es `_netdev`:** opción de `fstab` que dice "esto necesita red: esperá a que haya red antes de montar".
**Qué es `nofail`:** opción de `fstab` que dice "si no podés montar, seguí arrancando". Sin ella, un servidor NFS caído deja la VM trabada en el arranque (Día 6: el `fstab` roto).
**Qué es `systemctl daemon-reload` después de `fstab`:** systemd convierte cada línea de `fstab` en una unidad de montaje; después de editar el archivo hay que avisarle (Día 6).
**Qué es `mount -a`:** montar todo lo de `fstab` que no esté montado. Es la prueba antes de reiniciar: si la línea está mal, falla ahora y no en el arranque.
**Qué es `nfs_t`:** la etiqueta SELinux con la que el **cliente** ve lo que monta por NFS. Se mira, no se toca.
**Qué son `nfs_export_all_ro` / `nfs_export_all_rw`:** booleanos SELinux (Día 8) del servidor: "se puede exportar cualquier carpeta, en lectura / en escritura". Vienen encendidos: por eso exportar no pide `semanage fcontext`. `use_nfs_home_dirs` es para homes por NFS: apagado.
**Qué es autofs:** un servicio que vigila carpetas y monta lo que corresponde **cuando alguien entra**, y desmonta cuando pasa un tiempo sin uso. Evita montajes colgados y arranques lentos. El Día 6 se explicó el concepto; hoy se practica.
**Qué es el mapa maestro:** `/etc/auto.master`: qué carpetas vigilar y en qué archivo está el detalle de cada una. No se edita: se agregan archivos `.autofs` en `/etc/auto.master.d/`, que el maestro incluye con la línea `+dir:/etc/auto.master.d`.
**Qué es un mapa indirecto:** una carpeta padre (`/remoto`) que autofs controla entera. Cada línea del mapa es una **clave** (el nombre de una subcarpeta) con su origen. `ls /remoto` sale vacío hasta que alguien pide `/remoto/compartido` por su nombre exacto: es lo que más confunde.
**Qué es un mapa directo:** rutas completas (`/datos/nfs`), cada una montada donde dice, sin que autofs controle la carpeta padre. En el maestro se declara con `/-` en la primera columna. Sirve cuando el punto de montaje está en medio de otras cosas, como `/datos` del Día 6.
**Qué es `--timeout=60`:** segundos sin uso hasta desmontar. El valor por defecto es 300 (`/etc/autofs.conf`). autofs revisa cada cuarto del timeout, así que el desmontaje pasa entre 60 y 75 segundos después del último acceso.
**Qué es `automount`:** el programa que corre detrás del servicio `autofs`. `sudo automount -m` vuelca toda la configuración que cargó: sirve para ver si leyó bien los mapas.
**Qué es `systemctl reload autofs`:** releer los mapas sin cortar nada. Va después de cada edición de un `.autofs` o de un mapa.

---

## NFS: el disco de red

**Qué decir:** *"el servidor exporta, el cliente monta. Es el `mount` del Día 6, con la diferencia de que lo que se cuelga está en otra máquina."*
**Qué señalar:** un solo puerto, 2049. Y la tabla: el cliente no tiene servicio, solo `mount`.

---

## `/etc/exports`

Leer el diagrama señalando los tres pedazos.
**La frase que hay que decir:** *"sin espacio antes del paréntesis. Con un espacio, la línea dice 'esta carpeta para 192.168.56.0/24' y 'para todo el mundo, con opciones por defecto'. `exportfs -v` lo delata."*
**Qué señalar:** `no_root_squash` en `compartido` es para el laboratorio (somos servidor y cliente a la vez). En producción, nunca.

---

## Cliente: tres formas de montar

**Qué decir:** *"a mano se pierde al reiniciar; `fstab` es para siempre; autofs es cuando se usa. Hoy hacemos las tres, en ese orden."*
**Qué señalar:** en la línea de `fstab`, `_netdev` y `nofail`. Y la frase de abajo: `daemon-reload` y `mount -a` **antes** de reiniciar.

---

## autofs

Leer los tres archivos del diagrama de arriba a abajo: maestro → indirecto → directo.
**Qué decir:** *"el maestro dice 'vigilá `/remoto` con este archivo' y 'las rutas completas están en este otro'. El indirecto tiene subcarpetas; el directo, rutas completas."*
**La frase para el punto que confunde:** *"`ls /remoto` va a salir vacío y va a estar bien. autofs no muestra lo que no se pidió. Hay que escribir el nombre exacto: `ls /remoto/compartido`."*
**Qué señalar:** la fila de `reload`: después de editar un mapa, siempre.
