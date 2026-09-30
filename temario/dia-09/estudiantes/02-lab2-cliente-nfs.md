# Lab 2.2 — Cliente NFS y `fstab`

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

Ahora el segundo montaje: `127.0.0.1:/srv/nfs/lectura` en `/mnt/lectura`.

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

Después, intentar crear un archivo en `/mnt/lectura`, primero sin `sudo` y después con `sudo`.

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

**Comprobar:**
```
#192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
```

---

# Solución — todos los comandos

```bash
# Parte 1 — montar a mano los dos
sudo mkdir -p /mnt/nfs /mnt/lectura
sudo mount -t nfs 192.168.56.10:/srv/nfs/compartido /mnt/nfs
sudo mount -t nfs 127.0.0.1:/srv/nfs/lectura /mnt/lectura
mount | grep nfs4
df -hT /mnt/nfs /mnt/lectura

# Parte 2 — escribir, leer, y chocar
echo "escrito desde el cliente" > /mnt/nfs/prueba.txt
cat /mnt/nfs/prueba.txt
ls -l /srv/nfs/compartido/
touch /mnt/lectura/no-se-puede.txt        # Read-only file system
sudo touch /mnt/lectura/tampoco-root.txt  # Read-only file system, ni con sudo

# Parte 3 — la etiqueta SELinux del cliente
ls -Zd /mnt/nfs

# Parte 4 — permanente con fstab
sudo umount /mnt/nfs /mnt/lectura
echo "192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/nfs

# Parte 5 — comentar la línea (el próximo lab usa autofs)
sudo umount /mnt/nfs
sudo vim /etc/fstab
```
En la última línea, poner `#` al principio. Guardar con `:wq`.
```bash
tail -1 /etc/fstab
sudo systemctl daemon-reload
```

**El servidor manda, no el cliente.** En la Parte 1 `/mnt/lectura` aparece montado con `rw` — eso es lo que **pidió** el cliente. Cuando se intenta escribir, salta `Read-only file system`, porque el servidor lo exportó `ro`. Ni con `sudo` se puede: la restricción no está en los permisos locales.

**Los archivos salen con el mismo dueño en los dos lados** porque el UID de `student` es el mismo acá y allá (es la misma máquina). Entre servidores distintos, si los UID no coinciden, un archivo de `student` puede aparecer como otro usuario del otro lado. Eso es lo que resuelven LDAP o idmapd.

**Las dos opciones que salvan el arranque:**

| Opción | Qué hace |
|---|---|
| `_netdev` | esperá a que haya red antes de montar |
| `nofail` | si el servidor no responde, seguí arrancando igual |

Sin ellas, un servidor NFS caído deja la VM colgada 90 segundos y después en modo de emergencia. Y **siempre `mount -a` antes de reiniciar**: un error en `fstab` se ve ahora y no en el próximo arranque.

**`nfs_t`** es la etiqueta con la que el cliente ve todo lo montado por NFS, sin importar qué etiqueta tenga del lado del servidor.

**Se comenta la línea, no se borra,** porque el próximo lab monta lo mismo con autofs y dos mecanismos peleando por el mismo punto de montaje dan errores raros.
