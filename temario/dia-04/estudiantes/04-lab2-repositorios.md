# Lab 2 — Repositorios: Red Hat, CodeReady Builder y EPEL

Vamos a ver cómo están definidos los repositorios, prender CodeReady Builder, instalar EPEL, instalar `htop` desde ahí, y dejar EPEL bajo control.

---

## Parte 1 — Los repositorios definidos

**¿Quién escribe `redhat.repo`, y por qué no se edita?**

```bash
ls /etc/yum.repos.d/
head -10 /etc/yum.repos.d/redhat.repo
sudo subscription-manager repos --list-enabled
sudo dnf repolist --all | grep codeready
```

Foto.

**Comprobar:**
```
redhat.repo
#
# Certificate-Based Repositories
# Managed by (rhsm) subscription-manager
#
# *** This file is auto-generated.  Changes made here will be over-written. ***
# *** Use "subscription-manager repo-override --help" if you wish to make changes. ***
#

[rhel-9-for-x86_64-baseos-rpms]
name = Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
+----------------------------------------------------------+
    Available Repositories in /etc/yum.repos.d/redhat.repo
+----------------------------------------------------------+
Repo ID:   rhel-9-for-x86_64-baseos-rpms
...
Repo ID:   rhel-9-for-x86_64-appstream-rpms
...
codeready-builder-for-rhel-9-x86_64-rpms   Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)   disabled
```
Lo genera `subscription-manager`; se sobreescribe solo. Hay decenas de repos, casi todos apagados.

---

## Parte 2 — Prender CodeReady Builder

**¿Cómo se prende un repositorio de Red Hat?**

```bash
arch
sudo subscription-manager repos --enable codeready-builder-for-rhel-9-x86_64-rpms
sudo dnf repolist
```

Si `arch` dijo `aarch64` (UTM), el nombre es `codeready-builder-for-rhel-9-aarch64-rpms`. Foto.

**Comprobar:**
```
x86_64
Repository 'codeready-builder-for-rhel-9-x86_64-rpms' is enabled for this system.
Updating Subscription Management repositories.
repo id                                   repo name
codeready-builder-for-rhel-9-x86_64-rpms  Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
```
EPEL depende de librerías que están en CodeReady Builder.

---

## Parte 3 — Instalar EPEL

**¿Qué trae exactamente el paquete `epel-release`?**

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
ls /etc/yum.repos.d/
grep enabled /etc/yum.repos.d/epel.repo
rpm -qi gpg-pubkey | grep Summary
```

Foto.

**Comprobar:**
```
Installing:
 epel-release      noarch   9-7.el9      @commandline    19 k
...
Complete!
epel.repo  epel-testing.repo  redhat.repo
enabled=1
enabled=0
enabled=0
Summary     : Red Hat, Inc. (release key 2) <security@redhat.com> public key
Summary     : Red Hat, Inc. (auxiliary key 3) <security@redhat.com> public key
```
`@commandline` = se instaló desde una URL, no desde un repo. Trae solo los `.repo` y la llave en `/etc/pki/rpm-gpg/`; la llave **todavía no está importada**: solo aparecen las dos de Red Hat.

---

## Parte 4 — Instalar `htop` desde EPEL

**¿Qué pasa la primera vez que se instala algo de un repositorio nuevo?**

```bash
sudo dnf install -y htop
rpm -qi gpg-pubkey | grep Summary
dnf info htop | grep repo
```

Después, `htop` medio minuto: `F6` ordenar, `F5` árbol, `q` salir. Foto.

**Comprobar:**
```
Extra Packages for Enterprise Linux 9 - x86_64              2.1 MB/s |  24 MB     00:11
...
Installing:
 htop       x86_64    3.3.0-1.el9      epel          185 k
...
Importing GPG key 0x3228467C:
 Userid     : "Fedora (epel9) <epel@fedoraproject.org>"
 Fingerprint: FF8A D134 4597 106E CE81 3B91 8A38 72BF 3228 467C
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-9
Key imported successfully
...
Complete!
Summary     : Red Hat, Inc. (release key 2) <security@redhat.com> public key
Summary     : Red Hat, Inc. (auxiliary key 3) <security@redhat.com> public key
Summary     : Fedora (epel9) <epel@fedoraproject.org> public key
From repo    : epel
```
Con `-y` la llave se importa sin preguntar; sin `-y`, `dnf` muestra la huella y espera: ese es el momento de compararla con la publicada. `From repo : epel` es la respuesta a "¿de dónde salió esto?".

---

## Parte 5 — EPEL bajo control

**¿Cómo dejo EPEL instalado pero apagado, y lo uso solo cuando quiero?**

```bash
sudo dnf config-manager --set-disabled epel
dnf repolist --all | grep epel
dnf info fail2ban
dnf --enablerepo=epel info fail2ban | head -6
sudo dnf config-manager --set-enabled epel
dnf repolist | grep epel
```

Foto.

**Comprobar:**
```
epel                        Extra Packages for Enterprise Linux 9 - x86_64   disabled
...
Error: No matching Packages to list
...
Available Packages
Name         : fail2ban
...
epel                        Extra Packages for Enterprise Linux 9 - x86_64
```
Apagado, `fail2ban` "no existe"; con `--enablerepo=epel` solo para ese comando, sí. En la VM del curso lo dejamos prendido; en un servidor de la institución, apagado.
