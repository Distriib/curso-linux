# 4 — SSH a fondo

SSH tiene dos partes: el **servidor** (`sshd`, corriendo en la VM, puerto 22) y el **cliente** (`ssh`, en su computadora y también dentro de la VM).

## Entrar, y ejecutar sin entrar

```bash
ssh student@192.168.56.10
ssh student@192.168.56.10 hostname
```

La segunda forma ejecuta un comando y sale. Es la base de la automatización remota.

## Claves en vez de contraseña

Un par de claves:

| Archivo | Qué es | Dónde vive |
|---|---|---|
| `~/.ssh/id_ed25519` | la **privada** | solo en su computadora. **Nunca se copia a ningún lado** |
| `~/.ssh/id_ed25519.pub` | la **pública** | se copia al servidor, a `~/.ssh/authorized_keys` |

```bash
ssh-keygen -t ed25519 -C "curso-rhel"
ssh-copy-id -p 2222 student@localhost
```

`ed25519` es el algoritmo recomendado hoy: corto, rápido y seguro.

**SSH es estricto con los permisos.** Si no son estos, ignora la clave sin avisar:

| Archivo | Permisos |
|---|---|
| `~/.ssh` | `700` |
| `~/.ssh/authorized_keys` | `600` |
| `~/.ssh/id_ed25519` | `600` |

## Alias de conexión

En **su computadora**, el archivo `~/.ssh/config`:

```
Host rhel01
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519
```

Y a partir de ahí, en vez de escribir todo:
```bash
ssh rhel01
```
El alias también funciona con `scp` y `rsync`.

## La huella del servidor

La primera vez que se conectan, `ssh` pregunta si confían en el servidor y guarda su huella en `~/.ssh/known_hosts`. Si algún día cambia, avisa con `REMOTE HOST IDENTIFICATION HAS CHANGED`.

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # la huella, vista desde el servidor
ssh-keygen -R 192.168.56.10                        # olvidar la huella guardada
```

## Copiar archivos

```bash
scp archivo.txt rhel01:documentos/
scp rhel01:/etc/hostname ./copia.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
```

| Comando | Cuándo |
|---|---|
| `scp` | copiar uno o pocos archivos |
| `rsync` | sincronizar carpetas: solo viaja lo que cambió |

Opciones de `rsync`: `-a` conserva permisos y fechas · `-v` cuenta lo que hace · `-z` comprime · `--delete` borra en el destino lo que ya no está en el origen.

> **La barra final importa:** `~/empresa/` copia el contenido; `~/empresa` sin barra crearía `empresa-copia/empresa/`.

> En `scp` el puerto es `-P` mayúscula; en `ssh` es `-p` minúscula.

## Un túnel

```bash
ssh -N -L 8081:localhost:8000 rhel01
```

"Lo que llegue a mi puerto `8081`, mandalo por SSH hasta la VM y entregalo a su `localhost:8000`". Sirve para llegar a servicios internos del servidor sin abrir puertos en el firewall. `-N` = no abrir una shell.
