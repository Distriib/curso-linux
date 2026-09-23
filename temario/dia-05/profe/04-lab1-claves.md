# Lab — Claves SSH (comandos)

## Parte 1 — en el equipo del participante
```bash
ssh-keygen -t ed25519 -C "curso-rhel"
```

## Parte 2 — en el equipo del participante
Mac/Linux:
```bash
ssh-copy-id -p 2222 student@localhost
```
Windows:
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 student@localhost "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## Parte 3 — en el equipo del participante
```bash
ssh -p 2222 student@localhost hostname
ssh student@192.168.56.10 hostname
```

## Parte 4 — dentro de la VM
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "student@rhel01"
ssh-copy-id student@localhost
ssh localhost hostname
```

## Parte 5
```bash
ls -ld ~/.ssh
ls -l ~/.ssh
cat ~/.ssh/authorized_keys
```

## Parte 6
```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

---

## Qué señalar

- **Parte 1:** los dos archivos que se crearon. *"El que termina en `.pub` se reparte. El otro no sale nunca de acá."*
- **Parte 2:** la contraseña se escribe **por última vez**. A partir de ahora, clave.
- **Parte 3:** es el momento del lab. Entra sin preguntar nada. Y la forma `ssh host comando` ejecuta y sale — es lo que usan los scripts para trabajar contra otros servidores.
- **Parte 4:** el `-N ""` es "sin passphrase" y el `-f` es el nombre del archivo; así no pregunta nada. Se hace dentro de la VM para que los de Windows también practiquen `ssh-copy-id`.
- **Parte 5:** las dos claves en `authorized_keys`, una por línea. Y los permisos: *"si copiaron la clave a mano sin `chmod`, acá se ve y se arregla"*.
- **Parte 6:** la huella del servidor tiene que ser **la misma** que `ssh` les mostró la primera vez que se conectaron. Eso es "verificar la huella", y es lo que protege de que alguien se haga pasar por el servidor.

## Si alguien sigue con contraseña

En este orden:
1. `ls -ld ~/.ssh` → tiene que ser `drwx------`
2. `ls -l ~/.ssh/authorized_keys` → tiene que ser `-rw-------`
3. Que la clave esté de verdad: `cat ~/.ssh/authorized_keys`
4. Desde el cliente, con `-v`, ver si ofrece la clave:
   `ssh -v -p 2222 student@localhost exit`

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| Sigue pidiendo contraseña | Permisos de `~/.ssh` o de `authorized_keys`. Es el 90% de los casos |
| `ssh-copy-id: command not found` | Están en Windows; usar el comando largo |
| `Permission denied (publickey)` al entrar por la IP | Copiaron la clave por el NAT pero no es el mismo archivo… en realidad sí lo es: revisar que sea el mismo usuario `student` |
| Sobrescribieron la clave vieja | Si tenían una del Día 1 y respondieron `y`, la anterior se perdió. Rehacer `ssh-copy-id` |
| En la VM, `ssh localhost` pregunta la huella | Normal la primera vez: `yes` |
