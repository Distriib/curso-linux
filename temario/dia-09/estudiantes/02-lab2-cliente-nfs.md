# Lab — Cliente NFS y `fstab`

Vamos a montar las dos carpetas exportadas desde la misma VM, escribir en una, chocar contra la otra, y dejar una línea correcta en `/etc/fstab`.

| Export | Punto de montaje |
|---|---|
| `192.168.56.10:/srv/nfs/compartido` | `/mnt/nfs` |
| `127.0.0.1:/srv/nfs/lectura` | `/mnt/lectura` |

---

## Parte 1 — Montar a mano

**¿Cómo se monta una carpeta que está en otro servidor?**

```bash
sudo mkdir -p /mnt/nfs /mnt/lectura
sudo mount -t nfs 192.168.56.10:/srv/nfs/compartido /mnt/nfs
```

Ahora ustedes: la segunda, `127.0.0.1:/srv/nfs/lectura` en `/mnt/lectura`. Foto.

**Comprobar:**
```bash
mount | grep nfs4
df -hT /mnt/nfs /mnt/lectura
```
```
192.168.56.10:/srv/nfs/compartido on /mnt/nfs type nfs4 (rw,relatime,vers=4.2,...,addr=192.168.56.10)
127.0.0.1:/srv/nfs/lectura on /mnt/lectura type nfs4 (rw,relatime,vers=4.2,...,addr=127.0.0.1)
Filesystem                         Type  Size  Used Avail Use% Mounted on
192.168.56.10:/srv/nfs/compartido  nfs4  ...                    /mnt/nfs
127.0.0.1:/srv/nfs/lectura         nfs4  ...                    /mnt/lectura
```
`vers=4.2` sin haberlo pedido. El `rw` de `/mnt/lectura` es lo que pidió el cliente; el servidor lo exportó `ro` y el servidor manda.

---

## Parte 2 — Escribir, leer, y chocar

**¿Con qué dueño aparece en el servidor un archivo escrito desde el cliente?**

```bash
echo "escrito desde el cliente" > /mnt/nfs/prueba.txt
cat /mnt/nfs/prueba.txt
ls -l /srv/nfs/compartido/
```

Ahora ustedes: lo mismo, y después intenten crear un archivo en `/mnt/lectura`, primero sin `sudo` y después con `sudo`. Foto.

**Comprobar:**
```
escrito desde el cliente
-rw-r--r--. 1 student student ... prueba.txt
touch: cannot touch '/mnt/lectura/no-se-puede.txt': Read-only file system
touch: cannot touch '/mnt/lectura/tampoco-root.txt': Read-only file system
```
Mismo dueño en los dos lados porque el UID es el mismo. Ni root escribe en un export `ro`.

---

## Parte 3 — La etiqueta SELinux del cliente

**¿Con qué etiqueta ve el cliente lo que montó por NFS?**

```bash
ls -Zd /mnt/nfs
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
system_u:object_r:nfs_t:s0 /mnt/nfs
```

---

## Parte 4 — Permanente con `fstab`

**¿Qué dos opciones evitan que un servidor NFS caído trabe el arranque?**

```bash
sudo umount /mnt/nfs /mnt/lectura
echo "192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/nfs
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
Filesystem                         Size  Used Avail Use% Mounted on
192.168.56.10:/srv/nfs/compartido  ...                     /mnt/nfs
```
`mount -a` antes de reiniciar es la red de seguridad: un error en `fstab` se ve ahora, no en el arranque.

---

## Parte 5 — Dejar la línea comentada

**¿Cómo se desactiva una línea de `fstab` sin borrarla?**

A partir del próximo lab el montaje lo hace autofs, y no queremos dos mecanismos peleando por lo mismo.

```bash
sudo umount /mnt/nfs
sudo vim /etc/fstab
```
En la última línea, poner `#` al principio. Guardar (`:wq`).
```bash
tail -1 /etc/fstab
sudo systemctl daemon-reload
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
#192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
```
