# Lab — FTP enjaulado

Vamos a levantar un FTP donde cada usuario ve solo su home, y a ver cómo SELinux lo frena hasta que se lo permitimos.

| Qué | Valor |
|---|---|
| Servidor | `vsftpd`, puerto 21 |
| Usuario | `ana`, clave `Pgn.2026` |
| Jaula | su home, `/home/ana` |

---

## Parte 1 — Instalar y leer lo que viene

**¿Qué ya viene apagado, y qué falta?**

```bash
sudo dnf install -y vsftpd
grep -n anonymous_enable /etc/vsftpd/vsftpd.conf
grep -n chroot_local_user /etc/vsftpd/vsftpd.conf
```

Todos a la vez. Foto.

**Comprobar:**
```
12:anonymous_enable=NO
100:#chroot_local_user=YES
```
Anónimo apagado. El enjaulado está comentado: falta.

---

## Parte 2 — Enjaular y arrancar

**¿Por qué hacen falta dos líneas y no una?**

```bash
echo "chroot_local_user=YES" | sudo tee -a /etc/vsftpd/vsftpd.conf
echo "allow_writeable_chroot=YES" | sudo tee -a /etc/vsftpd/vsftpd.conf
sudo systemctl enable --now vsftpd
sudo firewall-cmd --add-service=ftp --permanent
sudo firewall-cmd --zone=internal --add-service=ftp --permanent
sudo firewall-cmd --reload
sudo ss -tlnp | grep vsftpd
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
chroot_local_user=YES
allow_writeable_chroot=YES
success
success
success
LISTEN 0 32 *:21 *:* users:(("vsftpd",...
```

---

## Parte 3 — SELinux lo frena

**La clave es correcta y el servicio está arriba. ¿Quién dice que no?**

```bash
curl ftp://localhost/ --user ana:Pgn.2026
sudo ausearch -m AVC -ts recent | grep vsftpd
getsebool ftp_home_dir
sudo setsebool -P ftp_home_dir on
curl ftp://localhost/ --user ana:Pgn.2026
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
curl: (67) Access denied: 500
type=AVC msg=audit(...): avc:  denied  { search } for  pid=... comm="vsftpd" ... tcontext=...:user_home_dir_t:s0 ...
ftp_home_dir --> off
```
El segundo `curl` ya no falla: devuelve la lista del home de `ana` (vacía, o con `mi-archivo.txt` del Día 3).

---

## Parte 4 — Subir un archivo

**¿Dónde cae en el servidor lo que se sube por FTP?**

```bash
curl -T /etc/hostname ftp://localhost/subido.txt --user ana:Pgn.2026
curl ftp://localhost/ --user ana:Pgn.2026
sudo ls -l /home/ana/
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
-rw-r--r--    1 2001     2001            7 ... subido.txt
-rw-r--r--. 1 ana ana 7 ... subido.txt
```
FTP muestra números (`2001`), no nombres. Cayó en el home de `ana`, que es toda su jaula.
