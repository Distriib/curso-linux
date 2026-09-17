# Lab 2.3 — Montaje permanente con `/etc/fstab`

Vamos a dejar `/archivos` montado en cada arranque, por UUID, y a probarlo **sin reiniciar**. Todos tipean cada comando; cuando dice **Ahora ustedes**, lo hacen solos. Foto de cada parte. `/archivos` quedó desmontado al final del lab anterior.

---

## Parte 1 — ¿Con qué UUID lo identifico?

```bash
sudo blkid -s UUID -o value /dev/sdb1
```

**Comprobar:**
```
c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90
```
El de cada uno es distinto. **Copiarlo**: seleccionar con el mouse en la terminal (se pega con clic derecho o `Ctrl+Shift+V`, según la terminal).

---

## Parte 2 — ¿Cómo queda la línea en `fstab`?

```bash
sudo vim /etc/fstab
```
`G` (última línea) → `o` (línea nueva) → escribir, pegando el UUID:
```
UUID=c81e7a22-0f3d-4b6e-9a77-2d5f8c1b3e90  /archivos  xfs  defaults,nofail  0 0
```
`Esc` → `:wq`.

```bash
tail -1 /etc/fstab
```

Ahora ustedes: la misma línea, con **su** UUID. Foto del `tail`.

**Comprobar:** `tail` muestra la línea con `UUID=`, `/archivos`, `xfs`, `defaults,nofail`, `0 0`. Seis campos.

---

## Parte 3 — ¿Cómo lo pruebo sin reiniciar?

```bash
sudo systemctl daemon-reload
sudo mount -a
findmnt /archivos
sudo findmnt --verify
```

**Comprobar:**
```
TARGET    SOURCE    FSTYPE OPTIONS
/archivos /dev/sdb1 xfs    rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota
Success, no errors or warnings detected
```
Nadie escribió `mount /dev/sdb1`: lo montó `mount -a` leyendo la línea nueva. Eso mismo hace el sistema al arrancar. Si `mount -a` dice `can't find UUID=...`: el UUID está mal copiado; `blkid` de nuevo y corregir en `vim`.

---

## Parte 4 — ¿Por qué no me deja desmontar?

```bash
cd /archivos
sudo umount /archivos
sudo fuser -vm /archivos
```

**Comprobar:**
```
umount: /archivos: target is busy.
                     USER        PID ACCESS COMMAND
/archivos:           student    2417 ..c.. bash
```
Alguien está **parado adentro**: tu propia shell (`c` = carpeta actual). Puede aparecer también `sudo` o `fuser`: los corriste desde adentro. Salir y desmontar:

```bash
cd ~
sudo umount /archivos
sudo mount -a
df -h /archivos
```

**Comprobar:** `umount` no dice nada (éxito) y `df` vuelve a mostrar `/dev/sdb1 ... /archivos`. Foto.
