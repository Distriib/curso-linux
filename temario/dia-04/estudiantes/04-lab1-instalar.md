# Lab 1 — Consultar, instalar, quitar y deshacer

Vamos a consultar la base RPM, instalar `wget` con `dnf`, quitar `tree` y volverlo a instalar desde su `.rpm` con `rpm`, y deshacer una transacción con `dnf history undo`.

---

## Parte 1 — `yum` es `dnf`; los repositorios; la base RPM

**¿Cuántos paquetes tiene instalada esta VM, y cuál fue el último?**

```bash
ls -l /usr/bin/yum /usr/bin/dnf
sudo dnf repolist
rpm -qa | wc -l
rpm -qa --last | head -3
```

Foto.

**Comprobar:**
```
lrwxrwxrwx. 1 root root 5 ... /usr/bin/dnf -> dnf-3
lrwxrwxrwx. 1 root root 5 ... /usr/bin/yum -> dnf-3
Updating Subscription Management repositories.
repo id                                   repo name
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
612
httpd-2.4.57-11.el9_4.x86_64                  Thu 04 Sep 2026 10:12:40 AM EST
httpd-core-2.4.57-11.el9_4.x86_64             Thu 04 Sep 2026 10:12:40 AM EST
...
```
Lo último fue `httpd`, hace un rato. En UTM los ids dicen `aarch64`.

---

## Parte 2 — Consultas sobre lo instalado

**¿De qué paquete salió `/usr/bin/ls`? ¿Y `monitor.sh`?**

```bash
rpm -qi bash | head -10
rpm -ql tree
rpm -qc openssh-server
rpm -qf /usr/bin/ls /etc/ssh/sshd_config /usr/local/bin/monitor.sh
rpm -V openssh-server
```

Ahora ustedes: `rpm -qi chrony | head -10`, `rpm -qc chrony`, `rpm -qf /usr/bin/passwd`. Foto.

**Comprobar:**
```
Name        : bash
Version     : 5.1.8
Release     : 9.el9
Architecture: x86_64
Install Date: ...
Group       : Unspecified
Size        : 7738298
License     : GPLv3+
Signature   : RSA/SHA256, ..., Key ID 199e2f91fd431d51
Source RPM  : bash-5.1.8-9.el9.src.rpm
/usr/bin/tree
/usr/lib/.build-id
...
/usr/share/man/man1/tree.1.gz
/etc/pam.d/sshd
/etc/ssh/sshd_config
/etc/ssh/sshd_config.d/50-redhat.conf
/etc/sysconfig/sshd
coreutils-8.32-35.el9.x86_64
openssh-server-8.7p1-38.el9.x86_64
file /usr/local/bin/monitor.sh is not owned by any package
```
`Key ID 199e2f91fd431d51` es la llave de Red Hat. `-qc` = "¿qué archivos puedo editar?". `monitor.sh` no es de ningún paquete: lo escribimos nosotros. `rpm -V` sin salida = nadie tocó los archivos.

---

## Parte 3 — Buscar, informarse, instalar

**¿Qué miro antes de instalar algo?**

```bash
dnf search wget
dnf info wget | head -12
sudo dnf install -y wget
rpm -q wget
```

Ahora ustedes: `dnf search nmap` y `dnf info nmap | head -12` (sin instalar). Foto.

**Comprobar:**
```
===================== Name Exactly Matched: wget =====================
wget.x86_64 : A utility for retrieving files using the HTTP or FTP protocols
Available Packages
Name         : wget
Version      : 1.21.1
Release      : 8.el9_4
Architecture : x86_64
Size         : 790 k
Source       : wget-1.21.1-8.el9_4.src.rpm
Repository   : rhel-9-for-x86_64-appstream-rpms
...
Installed:
  wget-1.21.1-8.el9_4.x86_64
Complete!
wget-1.21.1-8.el9_4.x86_64
```
`info` dice de qué repositorio viene y cuánto pesa **antes** de instalar.

---

## Parte 4 — Quitar `tree` y reinstalarlo desde el `.rpm`

**¿Qué hay debajo de `dnf`?**

```bash
sudo dnf remove -y tree
cd /tmp
dnf download tree
ls -l tree-*.rpm
rpm -qpi tree-*.rpm | head -12
rpm -K tree-*.rpm
sudo rpm -ivh tree-*.rpm
rpm -q tree
cd
```

Foto.

**Comprobar:**
```
Removed:
  tree-1.8.0-10.el9.x86_64
Complete!
tree-1.8.0-10.el9.x86_64.rpm                    1.3 MB/s |  56 kB     00:00
-rw-r--r--. 1 student student 57344 ... tree-1.8.0-10.el9.x86_64.rpm
Name        : tree
Version     : 1.8.0
Release     : 10.el9
...
Signature   : RSA/SHA256, ..., Key ID 199e2f91fd431d51
...
tree-1.8.0-10.el9.x86_64.rpm: digests signatures OK
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:tree-1.8.0-10.el9                ################################# [100%]
tree-1.8.0-10.el9.x86_64
```
`-qp` consulta el archivo sin instalarlo. `rpm -K`: intacto y firmado por Red Hat. Funcionó porque `tree` no necesita nada más; con `httpd`, `rpm -ivh` habría fallado con `Failed dependencies`. La forma correcta para un `.rpm` suelto es `sudo dnf install ./archivo.rpm`.

---

## Parte 5 — El historial, y deshacer

**¿Cómo deshago una instalación entera, con sus dependencias, sin adivinar qué quitar?**

```bash
dnf history | head -8
dnf history list wget
```

Anotar el **ID** de la fila cuya `Command line` dice `install -y wget` (acá es `13`). Con **ese número**:

```bash
dnf history info 13 | head -12
sudo dnf history undo -y 13
rpm -q wget
dnf history | head -4
```

Foto.

**Comprobar:**
```
ID     | Command line                    | Date and time    | Action(s)      | Altered
-----------------------------------------------------------------------------------
    14 | remove -y tree                  | 2026-09-04 11:31 | Removed        |    1
    13 | install -y wget                 | 2026-09-04 11:28 | Install        |    1
    12 | install -y httpd                | 2026-09-04 10:12 | Install        |   10
...
Transaction ID : 13
Begin time     : ...
User           : Student <student>
Return-Code    : Success
Command Line   : install -y wget
Packages Altered:
    Install wget-1.21.1-8.el9_4.x86_64 @rhel-9-for-x86_64-appstream-rpms
...
Removed:
  wget-1.21.1-8.el9_4.x86_64
Complete!
package wget is not installed
    15 | history undo -y 13              | 2026-09-04 11:34 | Removed        |    1
    14 | remove -y tree                  | 2026-09-04 11:31 | Removed        |    1
```
**El número es distinto en cada VM: se lee, no se copia.** `history info` dice quién, cuándo, con qué comando y desde qué repositorio (`@...`). `undo` crea una transacción nueva: el historial queda completo. El `rpm -ivh` de `tree` **no** aparece: `rpm` no pasa por `dnf`.

---

## Parte 6 — ¿Qué paquete me da tal comando?

```bash
dnf provides /usr/bin/ss
dnf provides '*/htop'
```

Foto.

**Comprobar:**
```
iproute-6.2.0-5.el9.x86_64 : Advanced IP routing and network device configuration tools
Repo        : rhel-9-for-x86_64-baseos-rpms
Matched from:
Filename    : /usr/bin/ss
Error: No Matches found
```
`ss` viene en `iproute`. `htop` **no existe** en BaseOS ni AppStream: lo resolvemos con EPEL en el lab que sigue.
