# Lab — Alias de conexión y huella del servidor

Dejar de escribir `ssh -p 2222 student@localhost` cada vez, y saber qué hacer cuando SSH avisa de que el servidor cambió.

> Todo este lab se hace en **su propia computadora**, no en la VM.

---

## Parte 1 — Crear los alias

**Mac o Linux:**
```bash
vim ~/.ssh/config
```
Pegar adentro:
```
Host rhel01
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519

Host rhel01-nat
    HostName localhost
    Port 2222
    User student
    IdentityFile ~/.ssh/id_ed25519
```
Y después:
```bash
chmod 600 ~/.ssh/config
```

**Windows:**
```powershell
notepad $env:USERPROFILE\.ssh\config
```
Pegar el mismo contenido y guardar. (El archivo se llama `config`, sin extensión.)

---

## Parte 2 — Usarlos

```bash
ssh rhel01 hostname
ssh rhel01-nat "ip -br a show enp0s8"
```

Ahora ustedes: entrar con `ssh rhel01`, mirar el prompt, y salir con `exit`. Foto.

**Comprobar:** el primero responde `rhel01.lab.local`; el segundo, la línea de la tarjeta con sus dos IPs. **Sin escribir usuario, puerto ni clave.**

En un puesto de administración real, este archivo tiene decenas de servidores con su puerto, su usuario y su clave.

---

## Parte 3 — Qué pasa cuando cambia la huella

**¿Qué hago si SSH me avisa de que el servidor cambió?**

Vamos a provocarlo: borramos la huella guardada y volvemos a entrar.

```bash
ssh-keygen -R 192.168.56.10
ssh rhel01 hostname
```

**Comprobar:** vuelve a preguntar
```
The authenticity of host '192.168.56.10' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
Responder `yes`.

**La huella que muestra tiene que ser la misma** que vieron en la VM con `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`. Eso es verificarla.

Cuando en la vida real aparezca `REMOTE HOST IDENTIFICATION HAS CHANGED`, el procedimiento es exactamente este: averiguar **por qué** cambió (¿se reinstaló el servidor? ¿se restauró un snapshot? ¿o alguien está suplantándolo?), y recién después `ssh-keygen -R`.

---

# Solución — todos los comandos

Todo en su propia computadora.

## Parte 1
```bash
vim ~/.ssh/config
```
Contenido:
```
Host rhel01
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519

Host rhel01-nat
    HostName localhost
    Port 2222
    User student
    IdentityFile ~/.ssh/id_ed25519
```
```bash
chmod 600 ~/.ssh/config
```
En Windows: `notepad $env:USERPROFILE\.ssh\config` y pegar lo mismo. El archivo se llama `config`, sin `.txt`.

## Parte 2
```bash
ssh rhel01 hostname
ssh rhel01-nat "ip -br a show enp0s8"
ssh rhel01
exit
```

## Parte 3
```bash
ssh-keygen -R 192.168.56.10
ssh rhel01 hostname
```
Responder `yes`. La huella que muestra tiene que ser la misma que da `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` dentro de la VM.
