# Lab 2.4 — Swap en partición

Vamos a activar `sdb2` como swap y dejarla permanente. Foto de cada parte. Donde dice **Ahora ustedes**, el comando no está escrito: hay que resolverlo. La solución está al final de la hoja. En UTM: `vdb2`.

---

## Parte 1 — ¿Cuánta swap tiene el servidor hoy?

```bash
swapon --show
free -m
```

**Comprobar:**
```
/dev/dm-1 partition   2G   0B   -2
Swap:           2047           0        2047
```
Una sola: los 2 GiB de `rhel-swap` que hizo el instalador (`dm-1` es su otro nombre).

---

## Parte 2 — ¿Cómo se formatea y activa una swap?

```bash
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
free -m
```

**Comprobar:**
```
Setting up swapspace version 1, size = 512 MiB (536866816 bytes)
no label, UUID=9e1f2a3b-...
/dev/dm-1 partition   2G   0B   -2
/dev/sdb2 partition 512M   0B   -3
Swap:           2559           0        2559
```
Dos swaps. `PRIO`: el sistema usa primero la de número más alto (`-2` antes que `-3`).

---

## Parte 3 — ¿Cómo la dejo permanente?

```bash
sudo blkid -s UUID -o value /dev/sdb2
sudo vim /etc/fstab
```
`G` → `o` → escribir, pegando el UUID:
```
UUID=9e1f2a3b-4c5d-6e7f-8a9b-0c1d2e3f4a5b  swap  swap  defaults  0 0
```
`Esc` → `:wq`.
```bash
sudo systemctl daemon-reload
tail -2 /etc/fstab
```

Ahora ustedes: la misma línea, con **su** UUID. Foto del `tail`.

**Comprobar:** las dos últimas líneas de `fstab` son la de `/archivos` y la de swap. En la de swap, el campo 2 y el 3 dicen `swap`.

---

## Parte 4 — ¿Cómo sé que la línea funciona, sin reiniciar?

```bash
sudo swapoff /dev/sdb2
swapon --show
sudo swapon -a
swapon --show
```

**Comprobar:** después de `swapoff` queda solo `/dev/dm-1`; después de `swapon -a` vuelve `/dev/sdb2`. `swapon -a` la activó leyendo `fstab`: la línea está bien. Foto.

---

# Solución — todos los comandos

```bash
# Parte 1 — qué swap hay hoy
swapon --show
free -m

# Parte 2 — formatear y activar
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
swapon --show
free -m

# Parte 3 — el UUID
sudo blkid -s UUID -o value /dev/sdb2
```

**Parte 3 (Ahora ustedes)** — agregar la línea a `fstab` con **su propio** UUID:
```bash
sudo vim /etc/fstab
```
`G` → `o` → pegar, reemplazando el UUID:
```
UUID=9e1f2a3b-4c5d-6e7f-8a9b-0c1d2e3f4a5b  swap  swap  defaults  0 0
```
`Esc` → `:wq`
```bash
sudo systemctl daemon-reload
tail -2 /etc/fstab
```

```bash
# Parte 4 — probar la línea sin reiniciar
sudo swapoff /dev/sdb2
swapon --show
sudo swapon -a
swapon --show
```
En la línea de swap, el campo 2 y el 3 dicen los dos `swap`: no hay punto de montaje. En UTM, `vdb2`.
