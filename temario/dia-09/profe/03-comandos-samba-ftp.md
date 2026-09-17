# Comandos — Samba y FTP (guía del instructor)

Cinco minutos, sin tipear. Tres cosas que tienen que quedar: (1) Samba es la carpeta compartida que se abre desde Windows; (2) el usuario existe en Linux **y** tiene una contraseña Samba aparte (`smbpasswd -a`); (3) dos capas de permisos, Samba y Linux, y la etiqueta `samba_share_t`. FTP: existe, viaja en texto plano, para lo nuevo se usa SFTP.

**Qué es SMB:** *Server Message Block*, el protocolo de carpetas compartidas de Windows. También se lo llama CIFS (el nombre viejo): por eso el tipo de montaje en Linux es `cifs`.
**Qué es Samba:** el programa que habla SMB desde Linux. Un servidor RHEL con Samba aparece en Windows como cualquier otra carpeta compartida: `\\192.168.56.10\compartido`.
**Qué es `smb` / `nmb`:** los dos servicios. `smb` sirve los archivos (puerto `445/tcp`; el 139 es compatibilidad). `nmb` responde por el nombre del servidor a equipos viejos; se habilita por costumbre.
**Qué es `samba-client`:** el paquete con `smbclient`, el cliente de línea de comandos.
**Qué es `cifs-utils`:** lo que permite `mount -t cifs`, montar una carpeta Samba como una carpeta más de Linux.
**Qué es `smb.conf`:** `/etc/samba/smb.conf`, la configuración. Secciones entre corchetes: `[global]` para todo el servidor, y una por carpeta compartida, con el nombre que se ve desde afuera.
**Qué es `workgroup`:** el nombre del grupo de trabajo de Windows. `PGN`. Solo importa para redes sin dominio.
**Qué es `security = user`:** cada persona que entra tiene que ser un usuario Linux existente, y además tener contraseña Samba. Es el único modo que usamos.
**Qué es `map to guest = Never`:** nada de invitados: sin usuario válido, afuera.
**Qué es `valid users = @sistemas`:** quién puede entrar a esa carpeta. La `@` significa "grupo Linux": cualquier miembro de `sistemas` (Día 3).
**Qué es `writable = yes`:** se puede escribir. `testparm` lo muestra como `read only = No`: son sinónimos.
**Qué es `browseable = yes`:** la carpeta aparece en la lista cuando alguien pregunta qué comparte el servidor.
**Qué es `create mask` / `directory mask`:** los permisos con los que nacen los archivos (`0664`) y las carpetas (`2775`) que se crean por Samba. Es el `umask` de Samba (Día 3).
**Qué es `testparm -s`:** revisa `smb.conf` y lo imprime ordenado. `Loaded services file OK` = sin errores. Es el `configtest` de Samba. `-s` = sin esperar que apretés Enter.
**Qué es `smbpasswd -a`:** dar de alta un usuario en Samba con su contraseña Samba. Samba guarda las contraseñas con otro formato que Linux (Windows lo exige), así que hay dos contraseñas: hoy usamos la misma para no confundir.
**Qué es `pdbedit -L`:** listar los usuarios que Samba conoce. `ana:2001:` = nombre y UID.
**Qué es `samba_share_t`:** la etiqueta SELinux de una carpeta que Samba puede compartir. Se declara con `semanage fcontext` y se aplica con `restorecon -Rv`, igual que `httpd_sys_content_t` el Día 8.
**Qué es `"/srv/samba(/.*)?"`:** la forma de escribir "`/srv/samba` y todo lo que haya adentro" en `semanage fcontext`. Es la misma terminación que usaron el Día 8 con `/web`. Se copia tal cual, siempre igual: no hay que entenderla letra por letra.
**Qué es `smbclient -L`:** preguntarle al servidor qué comparte. Al final dice `Unable to connect with SMB1 -- no workgroup available`: SMB1 está apagado por inseguro; es normal.
**Qué es `smbclient //servidor/carpeta -U usuario`:** entrar a la carpeta. Adentro hay un prompt `smb: \>` con comandos parecidos a FTP: `ls`, `put` (subir), `get` (bajar), `exit`.
**Qué es `mount -t cifs`:** montar la carpeta Samba en Linux. `-o username=ana` = con qué usuario Samba entra; `uid=student,gid=sistemas` = como quién se **muestran** los archivos en el cliente. El servidor los escribe igual como `ana`, que fue quien se autenticó.
**Qué es un archivo de credenciales:** un archivo con `username=` y `password=`, permisos `600`, que `mount` lee con `credentials=`. Es la forma de dejar un montaje Samba en `fstab` sin escribir la clave ahí.
**Qué es FTP:** *File Transfer Protocol*. El protocolo viejo de transferir archivos. Usuario, clave y datos viajan **sin cifrar**. Sigue en impresoras, escáneres y sistemas que no hablan otra cosa.
**Qué es SFTP:** transferencia de archivos sobre SSH. Ya lo tienen: `sftp -P 2222 student@localhost`. Es lo que se recomienda para todo lo nuevo.
**Qué es `vsftpd`:** *very secure FTP daemon*, el servidor FTP de RHEL. Puerto 21. Servicio `ftp` en el firewall.
**Qué es anónimo:** entrar sin usuario. `anonymous_enable=NO` ya viene así.
**Qué es `chroot_local_user=YES`:** enjaular: el usuario ve su home como si fuera la raíz del servidor y no puede subir de ahí.
**Qué es `allow_writeable_chroot=YES`:** vsftpd se niega a enjaular en una carpeta donde el usuario puede escribir, salvo que se le diga explícitamente que sí. Como el home es escribible, sin esta línea da `500 OOPS`.
**Qué es `ftp_home_dir`:** booleano SELinux (Día 8): "vsftpd puede entrar a los home". Apagado por defecto. Sin él, la clave es correcta y el servidor está arriba, pero SELinux no deja entrar al home: `500 OOPS: cannot change directory`.
**Qué es `curl ftp://... --user u:c`:** `curl` también habla FTP. Sin más, lista la carpeta. `-T archivo` = subirlo (*transfer*).

---

## Samba

Leer la tabla de piezas y después el `smb.conf` de arriba a abajo, deteniéndose en `security = user`, `valid users = @sistemas` y los dos `mask`.
**Qué decir:** *"Samba mira dos veces: su propia lista (`valid users`, `writable`) y los permisos Linux de la carpeta. Si una de las dos dice no, es no. Por eso la carpeta es `root:sistemas 2775`, igual que `/srv/sistemas` del Día 3."*
**La frase para el punto que confunde:** *"la contraseña Samba es otra. El usuario Linux tiene que existir, pero además hay que darlo de alta con `smbpasswd -a`. Hoy usamos la misma clave en las dos para no marearnos."*
**Qué señalar:** en la tabla de clientes, la fila de Windows: `Win+R` y `\\192.168.56.10\compartido`. Es el final del lab y el momento que más les va a gustar.

---

## FTP

**Qué decir:** *"FTP viaja en texto plano: usuario, clave y datos. Lo vemos porque existe en equipos que no hablan otra cosa. Para todo lo nuevo, SFTP, que ya tienen con SSH."*
**Qué señalar:** las tres filas del medio de la tabla, en orden: enjaular (`chroot_local_user`), permitir la jaula escribible (`allow_writeable_chroot`), y el booleano SELinux. El lab es exactamente eso, y falla a propósito antes del booleano.
