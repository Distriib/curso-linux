# Lab 3 — Repositorio local desde la ISO

Vamos a hacer lo que hace un servidor **sin internet**: usar la ISO de instalación de RHEL 9 como repositorio, con verificación de firmas, e instalar desde ahí.

---

## Parte 1 — Conectar la ISO a la VM encendida

- **VirtualBox:** menú de la ventana de la VM → *Devices* → *Optical Drives* → *Choose a disk file...* → la ISO de RHEL 9 del Día 1.
- **UTM:** en la barra de la ventana de la VM, el icono de unidades → *CD/DVD* → *Change* → la ISO.

**¿Cómo sé que el sistema vio el disco?**

```bash
lsblk
sudo blkid /dev/sr0
```

Foto.

**Comprobar:**
```
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    0   20G  0 disk
├─sda1          8:1    0    1G  0 part /boot
└─sda2          8:2    0   19G  0 part
  ├─rhel-root 253:0    0   15G  0 lvm  /
  └─rhel-swap 253:1    0    4G  0 lvm  [SWAP]
sr0            11:0    1 10.6G  0 rom
/dev/sr0: BLOCK_SIZE="2048" UUID="..." LABEL="RHEL-9-4-0-BaseOS-x86_64" TYPE="iso9660"
```
`sr0` es el lector, `RM 1` = removible, `TYPE="iso9660"` = formato de CD/DVD. Si `sr0` no aparece, el hipervisor no conectó la ISO: repetir.

---

## Parte 2 — Montarla y reconocer su estructura

**¿Qué hay adentro de la ISO?**

```bash
sudo mkdir -p /mnt/dvd
sudo mount -o ro /dev/sr0 /mnt/dvd
ls /mnt/dvd
ls /mnt/dvd/BaseOS /mnt/dvd/AppStream
ls /mnt/dvd/BaseOS/Packages | wc -l
```

Foto.

**Comprobar:**
```
mount: /mnt/dvd: WARNING: source write-protected, mounted read-only.
AppStream  BaseOS  EFI  EULA  GPL  RPM-GPG-KEY-redhat-beta  RPM-GPG-KEY-redhat-release  extra_files.json  images  isolinux  media.repo
/mnt/dvd/AppStream:
Packages  repodata
/mnt/dvd/BaseOS:
Packages  repodata
1100
```
Dos repositorios completos (`Packages/` + `repodata/`) y la llave pública de Red Hat en la raíz, la misma que hay en `/etc/pki/rpm-gpg/`.

---

## Parte 3 — Definir el repositorio

```bash
sudo vim /etc/yum.repos.d/rhel9-dvd.repo
```

`i`, pegar, `Esc`, `:wq`:

```
[dvd-baseos]
name=RHEL 9 DVD - BaseOS
baseurl=file:///mnt/dvd/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[dvd-appstream]
name=RHEL 9 DVD - AppStream
baseurl=file:///mnt/dvd/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

```bash
sudo dnf repolist
```

Foto.

**Comprobar:**
```
repo id                                   repo name
codeready-builder-for-rhel-9-x86_64-rpms  Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)
dvd-appstream                             RHEL 9 DVD - AppStream
dvd-baseos                                RHEL 9 DVD - BaseOS
epel                                      Extra Packages for Enterprise Linux 9 - x86_64
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
```
`file:///` con tres barras. Aunque el medio sea local, `gpgcheck=1` verifica que cada paquete esté firmado por Red Hat.

---

## Parte 4 — Instalar **solo** desde el DVD

**¿Cómo instalo como si no hubiera internet?**

```bash
sudo dnf --disablerepo="*" --enablerepo="dvd-baseos,dvd-appstream" install -y tmux sysstat
dnf info tmux | grep repo
dnf info sysstat | grep repo
```

Foto.

**Comprobar:**
```
RHEL 9 DVD - BaseOS                                     45 MB/s | 2.3 MB     00:00
RHEL 9 DVD - AppStream                                  60 MB/s | 8.5 MB     00:00
...
Installing:
 sysstat           x86_64   12.5.4-7.el9     dvd-appstream   478 k
 tmux              x86_64   3.2a-4.el9       dvd-baseos      480 k
...
Complete!
From repo    : dvd-baseos
From repo    : dvd-appstream
```
Todo salió del DVD, dependencias incluidas. `tmux` y `sysstat` quedan para el resto del curso.

---

## Parte 5 — Dejarlo listo pero apagado

**¿Qué pasa si reinicio con el repo prendido y la ISO desconectada?**

```bash
sudo dnf config-manager --set-disabled dvd-baseos dvd-appstream
grep enabled /etc/yum.repos.d/rhel9-dvd.repo
sudo umount /mnt/dvd
```

Foto.

**Comprobar:**
```
enabled=0
enabled=0
```
Con el repo prendido y la ISO ausente, **todo** `dnf` falla con `Failed to download metadata for repo 'dvd-baseos'`. Por eso se apaga. En un servidor real sin internet, la ISO se copia al disco, el repo queda en `enabled=1` y es el único.
