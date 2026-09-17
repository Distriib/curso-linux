# Lab — Carpeta compartida con Samba (comandos)

## Parte 1 — Paquetes y usuario
```bash
sudo dnf install -y samba samba-client cifs-utils
id ana
echo 'Pgn.2026' | sudo passwd --stdin ana
```
Si a alguien le falta `ana` o el grupo (no hizo el Lab 2 del Día 3):
```bash
sudo groupadd -g 3001 sistemas
sudo useradd -m -u 2001 -c "Ana Rodriguez - Sistemas" -G sistemas ana
echo 'Pgn.2026' | sudo passwd --stdin ana
```
Si `ana` está pero no en `sistemas`: `sudo usermod -aG sistemas ana`.

## Parte 2 — La carpeta y su etiqueta
```bash
sudo mkdir -p /srv/samba/compartido
sudo chown root:sistemas /srv/samba/compartido
sudo chmod 2775 /srv/samba/compartido
sudo semanage fcontext -a -t samba_share_t "/srv/samba(/.*)?"
sudo restorecon -Rv /srv/samba
ls -ldZ /srv/samba/compartido
```
Decir: "la terminación `(/.*)?` es la misma del Día 8: 'esta carpeta y todo lo de adentro'. Se copia igual siempre."

## Parte 3 — `smb.conf`
```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo vim /etc/samba/smb.conf
```
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
Pegar el texto en el chat. Decir: "`writable = yes` sale como `read only = No`: es lo mismo dicho al revés."

## Parte 4 — Contraseña Samba, servicios y firewall
```bash
sudo smbpasswd -a ana
sudo pdbedit -L
sudo systemctl enable --now smb nmb
systemctl is-active smb nmb
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --zone=internal --add-service=samba --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep smbd
```
`smbpasswd` pide la clave dos veces: `Pgn.2026`.

## Parte 5 — Entrar desde Linux con `smbclient`
```bash
smbclient -L localhost -U ana
smbclient //localhost/compartido -U ana
```
Adentro:
```
ls
put /etc/hostname hostname.txt
ls
exit
```
```bash
ls -l /srv/samba/compartido/
```
Decir: "llegó con grupo `sistemas` por el `2` del `2775`, y con `664` por el `create mask`. Las dos capas trabajando juntas."

Si `NT_STATUS_LOGON_FAILURE`: falta `smbpasswd -a ana` o escribió otra clave. Si `NT_STATUS_ACCESS_DENIED` al escribir: `ls -ldZ /srv/samba/compartido` (etiqueta), `id ana` (grupo), `2775`. Si `ana` quedó bloqueada por intentos fallidos (faillock del Día 8): `sudo faillock --user ana --reset`.

## Parte 6 — Montarla como una carpeta más
```bash
sudo mkdir -p /mnt/smb
sudo mount -t cifs //localhost/compartido /mnt/smb -o username=ana,uid=student,gid=sistemas
mount | grep cifs
touch /mnt/smb/desde-cifs.txt
ls -l /mnt/smb/ /srv/samba/compartido/
sudo umount /mnt/smb
```
Decir: "en el montaje parecen de `student` porque se lo pedimos con `uid=`; en el servidor los escribió `ana`, que es quien entró. `vers=3.1.1`: SMB moderno."

## Parte 7 — Desde Windows
Ellos: `Win+R` → `\\192.168.56.10\compartido`, usuario `ana`, clave `Pgn.2026`, crear un archivo de texto. Vos, en el Mac: Finder → `Cmd+K` → `smb://<tu IP host-only>/compartido`.
```bash
ls -l /srv/samba/compartido/
```
Si Windows no conecta: `sudo firewall-cmd --zone=internal --list-services` tiene que incluir `samba`; si Windows recuerda credenciales viejas, en un `cmd`: `net use \\192.168.56.10 /delete`.
