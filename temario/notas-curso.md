# Notas del curso

Cosas transversales, que no pertenecen a una jornada concreta.

---

## ARM vs x86: qué se puede compartir y qué no

El instructor da clase desde un Mac con chip Apple, en **UTM**, con RHEL para
**aarch64** (ARM). Los participantes usan **VirtualBox** en Windows, con RHEL
para **x86_64**.

### No se puede compartir: imágenes de máquina virtual

Una VM ARM **no arranca** en un PC Intel o AMD. No es cuestión de configuración:
son arquitecturas distintas.

Consecuencia práctica: **el instructor no puede entregar una VM ya instalada como
plan B** para quien no logre instalar. Subirla a Drive no sirve de nada.

**La alternativa que sí funciona:** si el primer día alguno de los cuatro termina
rápido y bien, que exporte su máquina (*Archivo → Exportar servicio virtualizado*,
genera un `.ova`) y se la pase al que quedó atrás. Esa sí es x86_64 y sí arranca
en las demás máquinas.

### Sí se puede compartir: los scripts

Los scripts son texto plano y funcionan igual en las dos arquitecturas. Los
comandos (`systemctl`, `chmod`, `fallocate`, `passwd`, `restorecon`) hacen lo
mismo en ARM que en x86.

Cada participante corre el script en su propia máquina. Eso incluye los scripts
que rompen el sistema a propósito en las Jornadas 4, 5, 6, 8 y 10.

### La excepción: nombres de disco

| Entorno | Discos |
|---|---|
| VirtualBox (participantes) | `sda`, `sdb`, `sdc` |
| UTM (instructor) | `vda`, `vdb`, `vdc` |

La causa es el controlador de disco virtual: VirtualBox emula SATA (`sd`), UTM
usa VirtIO (`vd`).

> ⚠️ **Un script que escriba `/dev/sdb` a mano falla en la máquina del
> instructor, y al revés.**

Afecta a los scripts de la **Jornada 6** (almacenamiento) y la **Jornada 10**
(el `fstab` roto), que son los que tocan discos.

**Cómo evitarlo:** que los scripts no escriban el nombre del disco. Usar el
nombre del volumen LVM (`/dev/vg_datos/lv_backups`), que es igual en ambas, o
detectar el disco al vuelo.

Es el tipo de error que funciona perfecto en la prueba del instructor y falla
en clase.

---

## Otras diferencias visibles entre las dos pantallas

Conviene anunciarlas en voz alta la primera vez que aparecen, en vez de que
alguien note la discrepancia y se desconcierte.

| Qué | Instructor (UTM) | Participantes (VirtualBox) |
|---|---|---|
| Arquitectura (`uname -m`) | `aarch64` | `x86_64` |
| Discos | `vda`, `vdb` | `sda`, `sdb` |
| Interfaz de red | `enp0s1` | `enp0s3` |
| Guardar estado | Clonar la VM | Snapshot real |

Los comandos son idénticos en todo lo demás.
