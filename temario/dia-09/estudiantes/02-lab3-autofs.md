# Lab 2.3 — autofs: montar al usar

Vamos a dejar que las carpetas se monten solas cuando alguien entra, y se desmonten solas cuando nadie las usa.

| Ruta en el cliente | Tipo de mapa | Origen |
|---|---|---|
| `/remoto/compartido` | indirecto | `192.168.56.10:/srv/nfs/compartido` |
| `/remoto/lectura` | indirecto | `127.0.0.1:/srv/nfs/lectura` |
| `/datos/nfs` | directo | `192.168.56.10:/srv/nfs/compartido` |

---

## Parte 1 — Instalar y leer el mapa maestro

**¿Dónde van nuestros mapas sin tocar el archivo original?**

```bash
sudo dnf install -y autofs
grep auto.master.d /etc/auto.master
```

**Comprobar:**
```
# Include /etc/auto.master.d/*.autofs
+dir:/etc/auto.master.d
```
Cualquier archivo `.autofs` en esa carpeta se lee como parte del mapa maestro.

---

## Parte 2 — El mapa maestro del laboratorio

**¿Qué diferencia hay entre `/remoto` y `/-` en la primera columna?**

```bash
sudo vim /etc/auto.master.d/lab.autofs
```
Pegar:
```
/remoto     /etc/auto.remoto     --timeout=60
/-          /etc/auto.directo
```

**Comprobar:**
```bash
cat /etc/auto.master.d/lab.autofs
```
Dos líneas, tal cual.

---

## Parte 3 — Los dos mapas

**En el indirecto la primera columna es una subcarpeta; en el directo, ¿qué es?**

```bash
sudo vim /etc/auto.remoto
```
Pegar:
```
compartido    -rw,sync    192.168.56.10:/srv/nfs/compartido
lectura       -ro         127.0.0.1:/srv/nfs/lectura
```

```bash
echo "/datos/nfs    -rw,sync    192.168.56.10:/srv/nfs/compartido" | sudo tee /etc/auto.directo
```

**Comprobar:**
```bash
cat /etc/auto.remoto /etc/auto.directo
```
Tres líneas en total.

---

## Parte 4 — Arrancar y mirar lo que creó

**¿Por qué `ls /remoto` sale vacío si está funcionando?**

```bash
sudo systemctl enable --now autofs
systemctl is-active autofs
ls -ld /remoto /datos/nfs
ls /remoto
mount | grep autofs
```

**Comprobar:**
```
active
drwxr-xr-x. 2 root root 0 ... /remoto
drwxr-xr-x. 2 root root 0 ... /datos/nfs
/etc/auto.remoto on /remoto type autofs (rw,relatime,...,timeout=60,...,indirect,...)
/etc/auto.directo on /datos/nfs type autofs (rw,relatime,...,timeout=300,...,direct,...)
```
autofs creó las carpetas solo. `ls /remoto` no muestra nada: las subcarpetas aparecen cuando se piden por su nombre.

---

## Parte 5 — Disparar los montajes

**¿Quién ejecutó `mount`?**

```bash
ls /remoto/compartido
mount | grep nfs4
```

Después, `cat /remoto/lectura/README.txt` y `cat /datos/nfs/prueba.txt`.

**Comprobar:**
```
prueba.txt
192.168.56.10:/srv/nfs/compartido on /remoto/compartido type nfs4 ...
Solo lectura desde NFS - rhel01
escrito desde el cliente
```
Con `mount | grep nfs4` al final, tres montajes: `/remoto/compartido`, `/remoto/lectura` y `/datos/nfs`. Nadie ejecutó `mount`: entrar a la ruta lo hizo.

---

## Parte 6 — Se desmonta solo

**¿Cuánto tarda en soltarse una carpeta que nadie usa?**

```bash
cd ~
sleep 90
mount | grep nfs4
```

**Comprobar:** queda solo `/datos/nfs`. Los de `/remoto` tenían `--timeout=60` y se soltaron; el directo usa el valor por defecto, 300 segundos, y todavía sigue.

---

## Parte 7 — Cambios y diagnóstico

**¿Qué hay que hacer después de editar un mapa?**

```bash
sudo automount -m | head -20
sudo systemctl reload autofs
journalctl -u autofs --no-pager | tail -3
```

**Comprobar:** `automount -m` lista cada punto de montaje con su mapa y sus entradas. `reload` relee los mapas sin cortar nada: es lo que va después de cada edición.

---

# Solución — todos los comandos

```bash
# Parte 1 — instalar y leer el mapa maestro
sudo dnf install -y autofs
grep auto.master.d /etc/auto.master

# Parte 2 — el mapa maestro del laboratorio
sudo vim /etc/auto.master.d/lab.autofs
```
Contenido:
```
/remoto     /etc/auto.remoto     --timeout=60
/-          /etc/auto.directo
```
```bash
cat /etc/auto.master.d/lab.autofs

# Parte 3 — los dos mapas
sudo vim /etc/auto.remoto
```
Contenido:
```
compartido    -rw,sync    192.168.56.10:/srv/nfs/compartido
lectura       -ro         127.0.0.1:/srv/nfs/lectura
```
```bash
echo "/datos/nfs    -rw,sync    192.168.56.10:/srv/nfs/compartido" | sudo tee /etc/auto.directo
cat /etc/auto.remoto /etc/auto.directo

# Parte 4 — arrancar y mirar lo que creó
sudo systemctl enable --now autofs
systemctl is-active autofs
ls -ld /remoto /datos/nfs
ls /remoto                       # vacío, y está bien
mount | grep autofs

# Parte 5 — disparar los montajes (basta con entrar a la ruta)
ls /remoto/compartido
cat /remoto/lectura/README.txt
cat /datos/nfs/prueba.txt
mount | grep nfs4                # ahora sí, tres montajes

# Parte 6 — se desmonta solo
cd ~
sleep 90
mount | grep nfs4                # quedó solo /datos/nfs

# Parte 7 — cambios y diagnóstico
sudo automount -m | head -20
sudo systemctl reload autofs
journalctl -u autofs --no-pager | tail -3
```

**Tres archivos, dos niveles.** El mapa **maestro** (`/etc/auto.master.d/lab.autofs`) dice *qué carpeta vigilar* y *en qué archivo está el detalle*. Los mapas de abajo (`auto.remoto`, `auto.directo`) dicen *qué montar*.

**Indirecto contra directo** — la diferencia está en la primera columna del mapa maestro:

| Tipo | Mapa maestro | Mapa de abajo | Resultado |
|---|---|---|---|
| **Indirecto** | `/remoto` | `compartido  -rw  servidor:/ruta` | se monta en `/remoto/compartido` |
| **Directo** | `/-` | `/datos/nfs  -rw  servidor:/ruta` | se monta en la ruta absoluta `/datos/nfs` |

En el indirecto la primera columna del mapa de abajo es **una subcarpeta**; en el directo es **la ruta completa**, y por eso el maestro lleva `/-` en vez de una carpeta.

**`ls /remoto` sale vacío y está bien.** Las subcarpetas no existen hasta que alguien las pide por su nombre: `ls /remoto/compartido` es lo que dispara el `mount`. Nadie ejecutó `mount` a mano.

**El `--timeout` es por mapa.** `/remoto` tiene `--timeout=60` y se suelta al minuto sin uso; el directo no lo lleva y usa el valor por defecto, 300 segundos. Por eso en la Parte 6 sobrevive solo `/datos/nfs`.

**Después de editar cualquier mapa: `sudo systemctl reload autofs`.** Relee los archivos sin cortar los montajes que estén en uso.
