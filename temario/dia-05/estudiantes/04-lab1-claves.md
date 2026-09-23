# Lab — Entrar sin contraseña, con clave

Dejar de escribir la contraseña cada vez que entran a la VM.

---

## Parte 1 — Generar el par de claves

En **su propia computadora** (PowerShell en Windows, Terminal en Mac):

```bash
ssh-keygen -t ed25519 -C "curso-rhel"
```

Enter tres veces: la ruta por defecto está bien, y la passphrase se deja vacía.

**Comprobar:**
```
Your identification has been saved in /Users/ana/.ssh/id_ed25519
Your public key has been saved in /Users/ana/.ssh/id_ed25519.pub
```
Si dice que el archivo ya existe, responder `n` y usar el que ya tienen.

---

## Parte 2 — Copiar la clave pública a la VM

**Mac o Linux:**
```bash
ssh-copy-id -p 2222 student@localhost
```

**Windows (no existe `ssh-copy-id`, se hace lo mismo a mano):**
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 student@localhost "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Pide la contraseña **una última vez**.

**Comprobar:** `Number of key(s) added: 1`

---

## Parte 3 — Probar

```bash
ssh -p 2222 student@localhost hostname
```

**Comprobar:** responde `rhel01.lab.local` **sin pedir contraseña**.

Ahora ustedes: lo mismo pero por la IP, `ssh student@192.168.56.10 hostname`. Foto.

---

## Parte 4 — La misma operación, dentro de la VM

Para que todos lo practiquen (incluidos los de Windows), ahora dentro de la VM:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "student@rhel01"
ssh-copy-id student@localhost
ssh localhost hostname
```

**Comprobar:** la última línea responde el nombre del servidor sin pedir contraseña. (La primera vez pregunta por la huella: responder `yes`.)

---

## Parte 5 — Mirar los permisos

```bash
ls -ld ~/.ssh
ls -l ~/.ssh
cat ~/.ssh/authorized_keys
```

**Comprobar:**
```
drwx------. 2 student student ... /home/student/.ssh
-rw-------. 1 student student ... authorized_keys
-rw-------. 1 student student ... id_ed25519
-rw-r--r--. 1 student student ... id_ed25519.pub
```
En `authorized_keys` hay **dos** claves: la de su computadora y la de la propia VM.

Si los permisos no son esos, SSH ignora la clave **sin avisar** y sigue pidiendo contraseña. Se arregla:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## Parte 6 — La huella del servidor

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

**Comprobar:** una línea que empieza con `256 SHA256:...` — esa es la huella de **este** servidor. Es la misma que les mostró `ssh` la primera vez que se conectaron.

---

# Solución — todos los comandos

## Partes 1 a 3 — en su propia computadora
```bash
ssh-keygen -t ed25519 -C "curso-rhel"
```
Mac o Linux:
```bash
ssh-copy-id -p 2222 student@localhost
```
Windows (una sola línea):
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 student@localhost "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
```bash
ssh -p 2222 student@localhost hostname
ssh student@192.168.56.10 hostname
```

## Partes 4 a 6 — dentro de la VM
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "student@rhel01"
ssh-copy-id student@localhost
ssh localhost hostname

ls -ld ~/.ssh
ls -l ~/.ssh
cat ~/.ssh/authorized_keys

ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

## Si sigue pidiendo contraseña
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
