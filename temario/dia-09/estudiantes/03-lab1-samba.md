# Lab 3.1 — Carpeta compartida con Samba

Vamos a compartir `/srv/samba/compartido` con el grupo `sistemas`, entrar como `ana` desde Linux y desde Windows.

| Qué | Valor |
|---|---|
| Carpeta | `/srv/samba/compartido`, `root:sistemas`, `2775` |
| Recurso compartido | `compartido` |
| Usuario | `ana` (grupo `sistemas`), clave Linux y Samba `Pgn.2026` |

---

## Parte 1 — Paquetes y usuario

**¿Qué necesita un usuario para entrar por Samba?**

```bash
sudo dnf install -y samba samba-client cifs-utils
id ana
echo 'Pgn.2026' | sudo passwd --stdin ana
```

**Comprobar:**
```
uid=2001(ana) gid=2001(ana) groups=2001(ana),3001(sistemas)
Changing password for user ana.
passwd: all authentication tokens updated successfully.
```

---

## Parte 2 — La carpeta y su etiqueta

**¿Qué etiqueta SELinux necesita una carpeta que comparte Samba?**

```bash
sudo mkdir -p /srv/samba/compartido
sudo chown root:sistemas /srv/samba/compartido
sudo chmod 2775 /srv/samba/compartido
sudo semanage fcontext -a -t samba_share_t "/srv/samba(/.*)?"
sudo restorecon -Rv /srv/samba
ls -ldZ /srv/samba/compartido
```

**Comprobar:**
```
Relabeled /srv/samba from unconfined_u:object_r:var_t:s0 to unconfined_u:object_r:samba_share_t:s0
Relabeled /srv/samba/compartido from ... to unconfined_u:object_r:samba_share_t:s0
drwxrwsr-x. 2 root sistemas unconfined_u:object_r:samba_share_t:s0 ... /srv/samba/compartido
```

---

## Parte 3 — `smb.conf`

**¿Cómo se valida la configuración antes de arrancar?**

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo vim /etc/samba/smb.conf
```
Pegar:
```
[global]
    workgroup = PGN
    server string = Servidor de archivos rhel01
    security = user
    map to guest = Never
    log file = /var/log/samba/log.%m
    max log size = 50

[compartido]
    comment = Carpeta compartida del grupo sistemas
    path = /srv/samba/compartido
    valid users = @sistemas
    writable = yes
    browseable = yes
    create mask = 0664
    directory mask = 2775
```
```bash
testparm -s
```

**Comprobar:**
```
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
...
[compartido]
	comment = Carpeta compartida del grupo sistemas
	create mask = 0664
	directory mask = 02775
	path = /srv/samba/compartido
	read only = No
	valid users = @sistemas
```
`writable = yes` aparece como `read only = No`: es lo mismo.

---

## Parte 4 — Contraseña Samba, servicios y firewall

**¿Por qué hay que ponerle otra contraseña si ya tiene una en Linux?**

```bash
sudo smbpasswd -a ana
```
(`Pgn.2026` dos veces)
```bash
sudo pdbedit -L
sudo systemctl enable --now smb nmb
systemctl is-active smb nmb
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --zone=internal --add-service=samba --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep smbd
```

**Comprobar:**
```
Added user ana.
ana:2001:
active
active
success
success
success
LISTEN 0 50 0.0.0.0:139 ... users:(("smbd",...
LISTEN 0 50 0.0.0.0:445 ... users:(("smbd",...
```

---

## Parte 5 — Entrar desde Linux con `smbclient`

**¿Qué ve `ana` cuando entra?**

```bash
smbclient -L localhost -U ana
smbclient //localhost/compartido -U ana
```
(clave `Pgn.2026`). Adentro, en el prompt `smb: \>`:
```
ls
put /etc/hostname hostname.txt
ls
exit
```
```bash
ls -l /srv/samba/compartido/
```

**Comprobar:**
```
	Sharename       Type      Comment
	---------       ----      -------
	compartido      Disk      Carpeta compartida del grupo sistemas
	IPC...          IPC       IPC Service (Servidor de archivos rhel01)
...
putting file /etc/hostname as \hostname.txt ...
  hostname.txt                        A        7  ...
-rw-rw-r--. 1 ana sistemas 7 ... hostname.txt
```
Al final de `-L` dice `Unable to connect with SMB1 -- no workgroup available`: es normal, SMB1 está apagado. El archivo llegó con grupo `sistemas` (por el `2` de `2775`) y modo `664` (por `create mask`).

---

## Parte 6 — Montarla como una carpeta más

**¿Quién figura como dueño en el cliente, y quién en el servidor?**

```bash
sudo mkdir -p /mnt/smb
sudo mount -t cifs //localhost/compartido /mnt/smb -o username=ana,uid=student,gid=sistemas
```
(clave Samba de `ana`)
```bash
mount | grep cifs
touch /mnt/smb/desde-cifs.txt
ls -l /mnt/smb/ /srv/samba/compartido/
sudo umount /mnt/smb
```

**Comprobar:**
```
//localhost/compartido on /mnt/smb type cifs (rw,relatime,vers=3.1.1,...,username=ana,uid=1000,...,gid=3001,...)
/mnt/smb/:
-rw-rw-r--. 1 student sistemas 0 ... desde-cifs.txt
-rw-rw-r--. 1 student sistemas 7 ... hostname.txt
/srv/samba/compartido/:
-rw-rw-r--. 1 ana sistemas 0 ... desde-cifs.txt
-rw-rw-r--. 1 ana sistemas 7 ... hostname.txt
```
En el montaje parecen de `student` (por `uid=student`); en el servidor los escribió `ana`, que fue quien se autenticó.

---

## Parte 7 — Desde Windows

**¿Qué ve Windows cuando entra a la carpeta?**

En tu computadora: `Win+R`, escribir `\\192.168.56.10\compartido`, Enter. Usuario `ana`, clave `Pgn.2026`. Crear un archivo de texto adentro.

**Comprobar:**
```bash
ls -l /srv/samba/compartido/
```
El archivo creado desde Windows aparece como `ana sistemas`.

---

# Solución — todos los comandos

```bash
# Parte 1 — paquetes y usuario
sudo dnf install -y samba samba-client cifs-utils
id ana
echo 'Pgn.2026' | sudo passwd --stdin ana

# Parte 2 — la carpeta y su etiqueta SELinux
sudo mkdir -p /srv/samba/compartido
sudo chown root:sistemas /srv/samba/compartido
sudo chmod 2775 /srv/samba/compartido
sudo semanage fcontext -a -t samba_share_t "/srv/samba(/.*)?"
sudo restorecon -Rv /srv/samba
ls -ldZ /srv/samba/compartido

# Parte 3 — smb.conf
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo vim /etc/samba/smb.conf
```
Contenido:
```
[global]
    workgroup = PGN
    server string = Servidor de archivos rhel01
    security = user
    map to guest = Never
    log file = /var/log/samba/log.%m
    max log size = 50

[compartido]
    comment = Carpeta compartida del grupo sistemas
    path = /srv/samba/compartido
    valid users = @sistemas
    writable = yes
    browseable = yes
    create mask = 0664
    directory mask = 2775
```
```bash
testparm -s

# Parte 4 — contraseña Samba, servicios y firewall
sudo smbpasswd -a ana          # Pgn.2026 dos veces
sudo pdbedit -L
sudo systemctl enable --now smb nmb
systemctl is-active smb nmb
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --zone=internal --add-service=samba --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep smbd

# Parte 5 — entrar con smbclient
smbclient -L localhost -U ana
smbclient //localhost/compartido -U ana
```
Adentro del prompt `smb: \>`:
```
ls
put /etc/hostname hostname.txt
ls
exit
```
```bash
ls -l /srv/samba/compartido/

# Parte 6 — montarla como una carpeta más
sudo mkdir -p /mnt/smb
sudo mount -t cifs //localhost/compartido /mnt/smb -o username=ana,uid=student,gid=sistemas
mount | grep cifs
touch /mnt/smb/desde-cifs.txt
ls -l /mnt/smb/ /srv/samba/compartido/
sudo umount /mnt/smb

# Parte 7 — desde Windows: Win+R -> \\192.168.56.10\compartido
#            usuario ana, clave Pgn.2026, y crear un archivo adentro
ls -l /srv/samba/compartido/
```

**Dos contraseñas para el mismo usuario.** Samba guarda las suyas aparte, en su propia base (`pdbedit -L`), porque usa un formato de hash distinto al de `/etc/shadow`. Por eso además del `passwd` hace falta el `smbpasswd -a`. Si alguien cambia la de Linux, la de Samba **no** cambia sola.

**Las tres cosas que tienen que estar bien a la vez**, y cada una falla distinto:

| Capa | Qué se hizo | Si falta |
|---|---|---|
| Permisos Linux | `chown root:sistemas`, `chmod 2775` | entra pero no puede escribir |
| SELinux | `samba_share_t` sobre `/srv/samba` | entra y ve la carpeta vacía o da error de acceso |
| Firewall | servicio `samba` en las dos zonas | no conecta desde afuera |

**El `2` de `2775` es setgid:** todo lo que se cree adentro hereda el grupo `sistemas`, sin importar quién lo escriba. Junto con `create mask = 0664`, eso es lo que hace que el grupo entero pueda trabajar sobre los mismos archivos.

**Quién figura como dueño depende de dónde mires.** En `/mnt/smb` los archivos salen de `student` porque así se montó (`uid=student`); en `/srv/samba/compartido` salen de `ana`, que es quien realmente se autenticó. El montaje CIFS solo disfraza los dueños del lado del cliente.

**`Unable to connect with SMB1`** al final de `smbclient -L` es normal: SMB1 está apagado por inseguro.
