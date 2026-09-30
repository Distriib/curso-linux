# Lab 2.1 — Servidor NFS

Vamos a exportar dos carpetas: una con escritura para la red host-only y otra de solo lectura para la propia VM.

| Carpeta | Dueño | Para quién | Modo |
|---|---|---|---|
| `/srv/nfs/compartido` | `student` | `192.168.56.0/24` | `rw,sync,no_root_squash` |
| `/srv/nfs/lectura` | `root` | `127.0.0.1` | `ro,sync` |

---

## Parte 1 — Instalar y arrancar

**¿Por qué el servicio dice `active (exited)` si está funcionando?**

```bash
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server
systemctl status nfs-server --no-pager | head -4
sudo ss -tln | grep 2049
```

**Comprobar:**
```
● nfs-server.service - NFS server and services
     Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; ...)
     Active: active (exited) since ...
LISTEN 0 64 0.0.0.0:2049 0.0.0.0:*
```

---

## Parte 2 — Las carpetas

**¿Por qué `compartido` es de `student` y no de root?**

```bash
sudo mkdir -p /srv/nfs/compartido
sudo chown student:student /srv/nfs/compartido
```

Ahora la segunda carpeta: `/srv/nfs/lectura`, de root, con un archivo `README.txt` que diga `Solo lectura desde NFS - rhel01` (con `sudo tee`).

**Comprobar:**
```bash
ls -ld /srv/nfs/*
cat /srv/nfs/lectura/README.txt
```
```
drwxr-xr-x. 2 student student ... /srv/nfs/compartido
drwxr-xr-x. 2 root    root    ... /srv/nfs/lectura
Solo lectura desde NFS - rhel01
```

---

## Parte 3 — `/etc/exports`

**¿Cómo se dice "esta carpeta, para esta red, con estas opciones"?**

```bash
sudo vim /etc/exports
```
Pegar (sin espacio antes del paréntesis):
```
/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)
/srv/nfs/lectura       127.0.0.1(ro,sync)
```
```bash
sudo exportfs -rav
sudo exportfs -v
```

**Comprobar:**
```
exporting 192.168.56.0/24:/srv/nfs/compartido
exporting 127.0.0.1:/srv/nfs/lectura
/srv/nfs/compartido  192.168.56.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
/srv/nfs/lectura     127.0.0.1(sync,wdelay,hide,no_subtree_check,sec=sys,ro,secure,root_squash,no_all_squash)
```
`exportfs -v` muestra también las opciones por defecto que no escribimos.

---

## Parte 4 — Firewall y comprobación

**¿Qué tres servicios necesita NFS en el firewall?**

```bash
sudo firewall-cmd --permanent --add-service=nfs --add-service=rpc-bind --add-service=mountd
sudo firewall-cmd --permanent --zone=internal --add-service=nfs --add-service=rpc-bind --add-service=mountd
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
showmount -e localhost
```

**Comprobar:**
```
success
success
success
dhcpv6-client http https mountd nfs rpc-bind ssh
dhcpv6-client http https mdns mountd nfs rpc-bind samba-client ssh
Export list for localhost:
/srv/nfs/lectura    127.0.0.1
/srv/nfs/compartido 192.168.56.0/24
```

---

## Parte 5 — SELinux ya lo permite

**¿Por qué exportar no necesitó etiquetar nada?**

```bash
getsebool nfs_export_all_ro nfs_export_all_rw use_nfs_home_dirs
```

**Comprobar:**
```
nfs_export_all_ro --> on
nfs_export_all_rw --> on
use_nfs_home_dirs --> off
```
Los dos primeros vienen encendidos: el servidor puede exportar cualquier carpeta.

---

# Solución — todos los comandos

```bash
# Parte 1 — instalar y arrancar
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server
systemctl status nfs-server --no-pager | head -4
sudo ss -tln | grep 2049

# Parte 2 — las dos carpetas
sudo mkdir -p /srv/nfs/compartido
sudo chown student:student /srv/nfs/compartido
sudo mkdir -p /srv/nfs/lectura
echo "Solo lectura desde NFS - rhel01" | sudo tee /srv/nfs/lectura/README.txt
ls -ld /srv/nfs/*
cat /srv/nfs/lectura/README.txt

# Parte 3 — /etc/exports
sudo vim /etc/exports
```
Contenido (**sin espacio antes del paréntesis**):
```
/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)
/srv/nfs/lectura       127.0.0.1(ro,sync)
```
```bash
sudo exportfs -rav
sudo exportfs -v

# Parte 4 — firewall (TRES servicios) y comprobación
sudo firewall-cmd --permanent --add-service=nfs --add-service=rpc-bind --add-service=mountd
sudo firewall-cmd --permanent --zone=internal --add-service=nfs --add-service=rpc-bind --add-service=mountd
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
showmount -e localhost

# Parte 5 — SELinux ya lo permite
getsebool nfs_export_all_ro nfs_export_all_rw use_nfs_home_dirs
```

**Anatomía de una línea de `/etc/exports`:**
```
/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)
        │                    │          └ opciones, SIN espacio antes del paréntesis
        │                    └ para quién: una red, una IP, o un nombre
        └ qué carpeta se exporta
```
Un espacio antes del `(` cambia el significado: pasaría a exportarse para **todo el mundo** con las opciones por defecto. Es el error clásico.

| Opción | Qué hace |
|---|---|
| `rw` / `ro` | escritura / solo lectura — **manda el servidor**, no el cliente |
| `sync` | confirma la escritura recién cuando está en disco |
| `no_root_squash` | el root del cliente sigue siendo root en el servidor (peligroso; solo entre servidores de confianza) |
| `root_squash` (por defecto) | el root del cliente se convierte en `nobody` |

**`active (exited)` no es un error.** El servidor NFS vive en el kernel; la unidad de systemd solo lo prepara y termina. Lo que confirma que está arriba es `ss -tln | grep 2049`.

**Tres servicios en el firewall, no uno:** `nfs` (2049), `rpc-bind` (111) y `mountd`. Con solo `nfs` abierto, el cliente se queda colgado al montar.

**No hizo falta tocar SELinux** porque `nfs_export_all_ro` y `nfs_export_all_rw` vienen encendidos de fábrica.
