# Lab 4.1 — SSH solo con llaves, sin root, probado antes de cerrar la puerta

Vamos a endurecer `sshd` sin quedarnos afuera. La secuencia segura: (a) confirmar que la llave funciona, (b) **no cerrar** la sesión actual, (c) aplicar, (d) probar en una sesión **nueva**, (e) recién ahí cerrar la vieja.

| Dato | Valor |
|---|---|
| Ruta desde tu computadora | `ssh -p 2222 student@localhost` |
| Archivo nuevo | `/etc/ssh/sshd_config.d/50-hardening.conf` |
| Banner | `/etc/issue.net` |

---

## Parte 1 — (a) La llave funciona

**¿Entro sin contraseña?**

En una terminal de **tu computadora**:
```bash
ssh -p 2222 student@localhost hostname
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** `rhel01`, **sin** pedir contraseña. Si la pide: `ssh-copy-id -p 2222 student@localhost` y repetir. **No seguir hasta que esto funcione.**

---

## Parte 2 — Qué hay en la carpeta de fragmentos

**¿Dónde lee sshd su configuración, y en qué orden?**

En la VM:
```bash
grep -n Include /etc/ssh/sshd_config
ls -l /etc/ssh/sshd_config.d/
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
19:Include /etc/ssh/sshd_config.d/*.conf
-rw-------. 1 root root 719 ... 50-redhat.conf
```
Si además aparece `01-permitrootlogin.conf`, borrarlo: `sudo rm /etc/ssh/sshd_config.d/01-permitrootlogin.conf` (ordena antes que el nuestro y le ganaría).

---

## Parte 3 — El banner y el archivo de hardening

**¿Qué seis líneas cambian todo?**

```bash
echo "Sistema institucional. Acceso restringido a personal autorizado. Toda actividad es registrada." | sudo tee /etc/issue.net
sudo vim /etc/ssh/sshd_config.d/50-hardening.conf
```
Escribir estas líneas y guardar (`Esc` `:wq`):
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

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
-rw-------. 1 root root ... 50-hardening.conf
-rw-------. 1 root root 719 ... 50-redhat.conf
```

---

## Parte 4 — Validar antes de aplicar

**¿La sintaxis está bien? ¿Nuestro archivo ganó?**

```bash
sudo sshd -t
sudo sshd -T | grep -i permitrootlogin
sudo sshd -T | grep -i passwordauthentication
sudo sshd -T | grep -i allowusers
sudo sshd -T | grep -i banner
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
permitrootlogin no
passwordauthentication no
allowusers student
banner /etc/issue.net
```
`sshd -t` no imprimió nada: bien. `sshd -T` muestra lo que sshd **va a aplicar** después de leer todos los archivos: nuestro archivo ganó.

---

## Parte 5 — (c) Aplicar sin cerrar esta sesión

**¿Cómo aplico sin cortar mi propia conexión?**

```bash
sudo systemctl reload sshd
systemctl is-active sshd
```

Ahora ustedes: lo mismo. **No cierren esta ventana.** Foto.

**Comprobar:** `active`. Tu sesión sigue viva.

---

## Parte 6 — (d) Probar en una sesión nueva

**¿Entra quien debe, y no entra quien no?**

En una terminal **nueva** de tu computadora, tres pruebas:
```bash
ssh -p 2222 student@localhost hostname
ssh -p 2222 root@localhost
ssh -p 2222 -o PubkeyAuthentication=no student@localhost
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Sistema institucional. Acceso restringido a personal autorizado. Toda actividad es registrada.
rhel01
root@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
student@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```
El banner aparece **antes** de autenticar (en las tres pruebas). root ya no entra. Sin llave no hay forma: entre paréntesis no aparece `password`. (e) Ahora sí se puede cerrar la sesión vieja.
