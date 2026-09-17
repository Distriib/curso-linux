# Comandos — Samba y FTP

## Samba: carpetas compartidas para Windows

**SMB** es el protocolo de carpetas compartidas de Windows. **Samba** lo habla desde Linux: la carpeta se abre desde el Explorador de Windows con `\\192.168.56.10\compartido`.

| Pieza | Qué es |
|---|---|
| paquetes | `samba` (servidor), `samba-client` (`smbclient`), `cifs-utils` (montar desde Linux) |
| servicios | `smb` (archivos, puerto `445/tcp`) y `nmb` (nombres, compatibilidad con equipos viejos) |
| `/etc/samba/smb.conf` | la configuración |
| `testparm -s` | valida `smb.conf`: el `configtest` de Samba |
| firewall | servicio `samba`, en las dos zonas |
| SELinux | la carpeta lleva `samba_share_t` |

## `smb.conf` mínimo

```
[global]
    workgroup = PGN
    server string = Servidor de archivos rhel01
    security = user                  ← cada usuario Samba existe también como usuario Linux
    map to guest = Never             ← sin invitados
    log file = /var/log/samba/log.%m
    max log size = 50

[compartido]                         ← el nombre que se ve desde afuera
    comment = Carpeta compartida del grupo sistemas
    path = /srv/samba/compartido     ← la carpeta real
    valid users = @sistemas          ← @ = un grupo Linux
    writable = yes
    browseable = yes
    create mask = 0664               ← permisos de los archivos nuevos
    directory mask = 2775            ← permisos de las carpetas nuevas
```

Dos capas de permisos: **Samba** (`valid users`, `writable`) **y Linux** (dueño, grupo, modo). Si una niega, se niega. Por eso la carpeta es `root:sistemas 2775`, como `/srv/sistemas` del Día 3.

## Usuarios Samba

El usuario Linux existe, pero Samba guarda **su propia contraseña**.

| Comando | Qué hace |
|---|---|
| `sudo smbpasswd -a ana` | dar de alta a `ana` en Samba, con su contraseña Samba |
| `sudo pdbedit -L` | listar los usuarios Samba |

## Clientes

| Comando | Qué hace |
|---|---|
| `smbclient -L localhost -U ana` | listar qué comparte el servidor |
| `smbclient //localhost/compartido -U ana` | entrar a la carpeta; adentro: `ls`, `put archivo`, `get archivo`, `exit` |
| `sudo mount -t cifs //localhost/compartido /mnt/smb -o username=ana,uid=student,gid=sistemas` | montar la carpeta en Linux |
| `sudo mount -t cifs //srv/carpeta /mnt/x -o credentials=/etc/samba/cred-ana,uid=student` | igual, con usuario y clave en un archivo `600` (nunca la clave en `fstab`) |
| Windows: `Win+R` → `\\192.168.56.10\compartido` | usuario `ana`, clave `Pgn.2026` |

## FTP: existe, pero ya no se elige

FTP viaja **en texto plano**: usuario, clave y datos. Se enseña porque sigue en impresoras, escáneres y sistemas viejos. Para todo lo nuevo: **SFTP** (ya lo tienen con `sshd`) o HTTPS.

| Pieza | Qué es |
|---|---|
| `vsftpd` | el servidor FTP de RHEL; puerto `21/tcp`; servicio `ftp` en el firewall |
| `/etc/vsftpd/vsftpd.conf` | la configuración; ya viene con anónimo apagado |
| `chroot_local_user=YES` | **enjaular**: el usuario ve solo su home |
| `allow_writeable_chroot=YES` | necesario porque el home es escribible |
| `ftp_home_dir` | booleano SELinux: sin él, vsftpd no puede entrar a los home |

| Comando | Qué hace |
|---|---|
| `curl ftp://localhost/ --user ana:Pgn.2026` | listar |
| `curl -T /etc/hostname ftp://localhost/subido.txt --user ana:Pgn.2026` | subir (`-T` = transferir) |
| `sudo setsebool -P ftp_home_dir on` | permitirle los home, permanente |
