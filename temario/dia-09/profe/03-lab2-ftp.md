# Lab — FTP enjaulado (comandos)

## Parte 1 — Instalar y leer lo que viene
```bash
sudo dnf install -y vsftpd
grep -n anonymous_enable /etc/vsftpd/vsftpd.conf
grep -n chroot_local_user /etc/vsftpd/vsftpd.conf
```
Los números de línea pueden variar; importa `=NO` en el primero y el `#` delante en el segundo.

## Parte 2 — Enjaular y arrancar
```bash
echo "chroot_local_user=YES" | sudo tee -a /etc/vsftpd/vsftpd.conf
echo "allow_writeable_chroot=YES" | sudo tee -a /etc/vsftpd/vsftpd.conf
sudo systemctl enable --now vsftpd
sudo firewall-cmd --add-service=ftp --permanent
sudo firewall-cmd --zone=internal --add-service=ftp --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep vsftpd
```
Decir: "la primera línea enjaula; la segunda hace falta porque el home es escribible y vsftpd, si no, se niega con `500 OOPS`."

## Parte 3 — SELinux lo frena
```bash
curl ftp://localhost/ --user ana:Pgn.2026
sudo ausearch -m AVC -ts recent | grep vsftpd
getsebool ftp_home_dir
sudo setsebool -P ftp_home_dir on
curl ftp://localhost/ --user ana:Pgn.2026
```
Decir: "clave correcta, servicio arriba, y no entra. El `ausearch` dice quién: `vsftpd` quiso entrar a `user_home_dir_t` y SELinux dijo no. El método del Día 8: servicio → log → SELinux → booleano."

Si en la VM no existe el booleano `ftp_home_dir`: `sudo setsebool -P ftpd_full_access on` (más permisivo, sirve igual). Si `ana` quedó bloqueada por intentos fallidos: `sudo faillock --user ana --reset`.

## Parte 4 — Subir un archivo
```bash
curl -T /etc/hostname ftp://localhost/subido.txt --user ana:Pgn.2026
curl ftp://localhost/ --user ana:Pgn.2026
sudo ls -l /home/ana/
```
Decir: "FTP muestra `2001 2001`: números, no nombres. Y cayó en `/home/ana`, que es toda la jaula. Cerramos: para lo nuevo, SFTP."
