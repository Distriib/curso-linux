# Lab — autofs: montar al usar

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

Todos a la vez. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo, y después `cat /remoto/lectura/README.txt` y `cat /datos/nfs/prueba.txt`. Foto.

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

Ahora ustedes: lo mismo. Foto.

**Comprobar:** queda solo `/datos/nfs`. Los de `/remoto` tenían `--timeout=60` y se soltaron; el directo usa el valor por defecto, 300 segundos, y todavía sigue.

---

## Parte 7 — Cambios y diagnóstico

**¿Qué hay que hacer después de editar un mapa?**

```bash
sudo automount -m | head -20
sudo systemctl reload autofs
journalctl -u autofs --no-pager | tail -3
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** `automount -m` lista cada punto de montaje con su mapa y sus entradas. `reload` relee los mapas sin cortar nada: es lo que va después de cada edición.
