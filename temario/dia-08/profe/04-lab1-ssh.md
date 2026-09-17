# Lab — SSH solo con llaves, sin root, probado antes de cerrar la puerta (comandos)

Vos primero, ellos después, foto. **Antes de la Parte 3, nadie sigue si la Parte 1 le pidió contraseña.** Tené visible la ventana de la consola de tu VM por si hay que mostrar la salida de emergencia.

Salida de emergencia si alguien se queda afuera: consola de la VM (VirtualBox/UTM), entrar como `student`, y:
```bash
sudo rm /etc/ssh/sshd_config.d/50-hardening.conf
sudo systemctl reload sshd
```

## Parte 1 — (a) La llave funciona
En la computadora de cada uno (PowerShell o Terminal):
```bash
ssh -p 2222 student@localhost hostname
```
Si pide contraseña: `ssh-copy-id -p 2222 student@localhost` (en Windows no existe: copiar la llave a mano como el Día 5) y repetir. No seguir hasta que responda `rhel01` sin preguntar nada.

## Parte 2 — Qué hay en la carpeta de fragmentos
En la VM:
```bash
grep -n Include /etc/ssh/sshd_config
ls -l /etc/ssh/sshd_config.d/
```
Si aparece `01-permitrootlogin.conf` (lo crea el instalador cuando se marcó "permitir root por SSH"): `sudo rm /etc/ssh/sshd_config.d/01-permitrootlogin.conf`. Ordena antes que el nuestro y su `PermitRootLogin yes` ganaría.

## Parte 3 — El banner y el archivo de hardening
```bash
echo "Sistema institucional. Acceso restringido a personal autorizado. Toda actividad es registrada." | sudo tee /etc/issue.net
sudo vim /etc/ssh/sshd_config.d/50-hardening.conf
```
Contenido (`i`, escribir, `Esc`, `:wq`):
```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
AllowUsers student
Banner /etc/issue.net
```
```bash
sudo chmod 600 /etc/ssh/sshd_config.d/50-hardening.conf
ls -l /etc/ssh/sshd_config.d/
```
El `chmod 600` no es obligatorio para sshd; es la costumbre (el de Red Hat también es 600).

## Parte 4 — Validar antes de aplicar
```bash
sudo sshd -t
sudo sshd -T | grep -i permitrootlogin
sudo sshd -T | grep -i passwordauthentication
sudo sshd -T | grep -i allowusers
sudo sshd -T | grep -i banner
```
Si `sshd -t` imprime algo, es un error de tipeo en el archivo: dice archivo y línea (`Bad configuration option`). Corregir con vim antes de seguir. Nadie hace `reload` con `sshd -t` quejándose.

## Parte 5 — (c) Aplicar sin cerrar esta sesión
```bash
sudo systemctl reload sshd
systemctl is-active sshd
```
Qué decir: "esta ventana no se cierra hasta el final del lab."

## Parte 6 — (d) Probar en una sesión nueva
En una terminal **nueva** de la computadora de cada uno:
```bash
ssh -p 2222 student@localhost hostname
ssh -p 2222 root@localhost
ssh -p 2222 -o PubkeyAuthentication=no student@localhost
```
La primera: banner y `rhel01`. Las otras dos: `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`. Qué decir: "entre paréntesis no dice `password`: no hay forma de entrar sin llave."
Si a alguien la primera le da `Permission denied`: la llave no estaba (no hizo bien la Parte 1) o no entró como `student`. Salida de emergencia de arriba, `ssh-copy-id`, y repetir desde la Parte 5.
