# Reto — Ticket #2026-0614

**Solos, sin ayuda.** Si no se termina en clase, queda de tarea.

> **Volumen para respaldos y swap adicional**
> Solicitante: Equipo de Respaldos. Servidor: `rhel01`.

## Lo que piden

1. Un volumen lógico de **1 GiB** llamado **`lv_backups`** en el grupo **`vg_datos`**, con sistema de archivos **ext4**, montado de forma **permanente** en **`/backups`**. Si ese disco fallara, el servidor tiene que **seguir arrancando**.

2. **512 MiB de swap adicional** usando un **archivo** `/swapfile`, también permanente.

3. **Reiniciar** el servidor y demostrar que `/backups` volvió montado y que las tres swaps están activas. Antes de reiniciar, `sudo findmnt --verify` tiene que decir `Success`. La sesión SSH se corta: esperar un minuto y reconectar. Si no vuelve, abrir la ventana de la VM: está en modo de emergencia, y los pasos para salir están al final del archivo `02-comandos-particiones-fstab`.

4. Después del reinicio, el equipo pide **500 MiB más** en `/backups`. Ya están copiando archivos: **no se puede desmontar**. Mostrar `df -h /backups` antes y después.

## Verificación

Pegar en el chat la salida completa de:

```bash
df -h /backups /datos /archivos /stratis
swapon --show
sudo lvs vg_datos
grep backups /etc/fstab
grep swapfile /etc/fstab
sudo findmnt --verify
ls -l /swapfile
```

Está bien si: `/backups` es `/dev/mapper/vg_datos-lv_backups` de `1.5G` y los otros tres siguen montados; `swapon --show` lista `/dev/dm-1`, `/dev/sdb2` y `/swapfile file 512M`; `lvs` muestra `lv_backups` de `<1.49g` o `1.50g`; la línea de `/backups` en `fstab` dice `ext4` y `nofail`; la de `/swapfile` dice `swap  swap`; `findmnt --verify` dice `Success`; `/swapfile` es `-rw-------` de `root`.
