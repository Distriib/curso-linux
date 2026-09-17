# Comandos — software: `rpm`, `dnf` y repositorios

## Un paquete RPM

Un archivo `.rpm` trae: los archivos a instalar y dónde van, la descripción, la lista de **dependencias** (qué otros paquetes necesita) y una **firma** de quien lo publicó. El nombre lo dice todo:

```
httpd-2.4.57-11.el9_4.x86_64.rpm
 │     │      │   │     └── arquitectura: x86_64, aarch64, noarch (sirve en todas)
 │     │      │   └──────── el9 = RHEL 9; el9_4 = publicado para 9.4
 │     │      └──────────── release: iteración del empaquetado de Red Hat (parches)
 │     └─────────────────── version: la del proyecto original
 └───────────────────────── nombre
```

En RHEL la versión casi no cambia en 10 años; cambia el **release**, porque Red Hat aplica los parches de seguridad sin cambiar de versión.

## `rpm` — consultar la base de datos de lo instalado

```bash
rpm -q bash
rpm -qf /usr/bin/ls
```

| Comando | Pregunta que responde |
|---|---|
| `rpm -q tree` | ¿está instalado, y qué versión? |
| `rpm -qa \| wc -l` | ¿cuántos paquetes hay? |
| `rpm -qa --last \| head -3` | ¿qué se instaló último? |
| `rpm -qi bash` | información del paquete |
| `rpm -ql tree` | ¿qué archivos puso? |
| `rpm -qc openssh-server` | ¿cuáles son sus archivos de configuración? |
| `rpm -qf /usr/bin/ls` | ¿de qué paquete salió este archivo? |
| `rpm -V openssh-server` | ¿alguien modificó sus archivos? (sin salida = intacto) |
| `rpm -qpi archivo.rpm` · `rpm -qpl archivo.rpm` | lo mismo, sobre un `.rpm` **sin instalar** |
| `rpm -K archivo.rpm` | ¿la firma es válida? |
| `sudo rpm -ivh archivo.rpm` | instalar un `.rpm` a mano. **No resuelve dependencias** |

## `dnf` — instalar resolviendo dependencias

`yum` es el mismo programa (`ls -l /usr/bin/yum`). Baja de los repositorios, resuelve dependencias, verifica firmas y anota cada **transacción**.

```bash
sudo dnf repolist
```

| Comando | Qué hace |
|---|---|
| `dnf search wget` | buscar por nombre y descripción |
| `dnf info wget` | de qué repositorio viene y cuánto pesa, **antes** de instalar |
| `sudo dnf install -y wget` | instalar (`-y` = sí a todo) |
| `sudo dnf remove -y tree` | quitar |
| `dnf download tree` | bajar el `.rpm` sin instalarlo |
| `dnf provides '*/htop'` | ¿qué paquete me da este comando? |
| `dnf history` | las transacciones, con su número (ID) |
| `dnf history list wget` | las que tocaron ese paquete |
| `dnf history info 13` | quién, cuándo, qué comando, qué paquetes |
| `sudo dnf history undo -y 13` | deshacer **exactamente** esa transacción |
| `dnf repolist` · `dnf repolist --all` | repositorios activos · todos |

## Repositorios

Un repositorio es una carpeta (local o por red) con paquetes y un índice. Se definen en `/etc/yum.repos.d/nombre.repo`:

```
[dvd-baseos]                      id
name=RHEL 9 DVD - BaseOS          nombre legible
baseurl=file:///mnt/dvd/BaseOS    dónde está (o https://...)
enabled=1                         1 activo, 0 apagado
gpgcheck=1                        verificar la firma de cada paquete
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release    con esta llave
```

```bash
ls /etc/yum.repos.d/
```

| Repositorio | Qué tiene | Cómo se activa |
|---|---|---|
| **BaseOS** | el sistema: kernel, systemd, bash | viene activo |
| **AppStream** | aplicaciones: httpd, php, nodejs | viene activo |
| **CodeReady Builder** | librerías de desarrollo; EPEL lo necesita | `sudo subscription-manager repos --enable ...` |
| **EPEL** | miles de paquetes de la comunidad que Red Hat no incluye (`htop`, `fail2ban`) | instalar `epel-release` |

- `redhat.repo` lo genera `subscription-manager`: **no se edita**. Los repos de Red Hat se prenden con `subscription-manager repos --enable`.
- Los de terceros se prenden y apagan con `sudo dnf config-manager --set-enabled epel` / `--set-disabled epel`, o solo para un comando: `dnf --enablerepo=epel install ...`.
- EPEL no lo soporta Red Hat. Regla razonable en la institución: instalado pero apagado, y activado por comando para paquetes concretos.

## Firmas

Cada paquete de Red Hat va firmado. La llave pública está en `/etc/pki/rpm-gpg/` e importada en la base RPM (`rpm -qi gpg-pubkey | grep Summary`). Con `gpgcheck=1`, `dnf` rechaza lo que no verifique. Un repositorio nuevo pide importar su llave la primera vez.

## Sin internet

Montar la ISO de instalación y usarla como repositorio: `baseurl=file:///mnt/dvd/BaseOS`. Es el caso de la institución, y es el Lab 3.
