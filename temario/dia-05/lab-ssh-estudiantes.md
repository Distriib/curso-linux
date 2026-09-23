# Lab — Entrar al servidor sin contraseña

Hasta ahora escribimos la contraseña cada vez que entramos por SSH. Vamos a reemplazarla por un par de archivos.

## Dónde se hace cada cosa

**Las claves se crean en SU COMPUTADORA (Windows), no en la VM.**

```
  SU COMPUTADORA (Windows)                    LA VM (RHEL)
  ┌─────────────────────────┐                ┌──────────────────────┐
  │                         │                │                      │
  │  id_ed25519             │                │                      │
  │  la PRIVADA:            │                │                      │
  │  se queda acá,          │                │                      │
  │  nunca sale             │                │                      │
  │                         │                │                      │
  │  id_ed25519.pub ────────┼─── se copia ──→│  authorized_keys     │
  │  la PÚBLICA             │                │                      │
  └─────────────────────────┘                └──────────────────────┘
           ↑                                            ↑
   desde acá nos conectamos                   acá entramos sin contraseña
```

La clave privada tiene que estar en la máquina **desde la que uno se conecta**. Por eso va en su Windows.

---

## Antes de empezar: ¿dónde estoy parado?

**Mirá el prompt.** Es la única forma de saberlo, y es donde más se equivoca todo el mundo:

```
PS C:\Users\Juan>          ← PowerShell, en MI COMPUTADORA
[student@rhel01 ~]$        ← la VM (Linux)
```

Si están adentro de la VM y tienen que estar en PowerShell: escriban `exit` hasta que el prompt cambie.

> Todos los comandos se escriben en **PowerShell**, salvo los pasos 5 y 6, que son adentro de la VM. Cada paso avisa dónde va.

---

## Paso 0 — Comprobar que tienen el cliente de SSH

```powershell
ssh -V
```

**Tiene que salir** algo como `OpenSSH_for_Windows_8.9p1, LibreSSL 3.4.3`.

Si dice que no reconoce el comando: *Configuración > Aplicaciones > Características opcionales > Agregar una característica > Cliente de OpenSSH*. Instalar y reabrir PowerShell.

---

## Paso 1 — Ver si ya tienen claves

```powershell
dir $env:USERPROFILE\.ssh
```

**Qué mirar:** si aparece un archivo llamado `id_ed25519`, ya tienen una clave de antes — avisen y sáltense el paso 2. Si dice que la ruta no existe, está bien: seguimos.

---

## Paso 2 — Crear el par de claves

```powershell
ssh-keygen -t ed25519 -C "curso-rhel"
```

Pregunta tres cosas. **Enter en las tres.**

| Pregunta | Qué hacer |
|---|---|
| `Enter file in which to save the key` | Enter — el nombre por defecto está bien |
| `Enter passphrase` | Enter — vacía, para el curso |
| `Enter same passphrase again` | Enter |

**Tiene que salir:**
```
Your identification has been saved in C:\Users\...\.ssh\id_ed25519
Your public key has been saved in C:\Users\...\.ssh\id_ed25519.pub
The key fingerprint is:
SHA256:Qm4k...
```

**Qué significa cada parte del comando:**

| Parte | Qué es |
|---|---|
| `ssh-keygen` | el programa que crea claves |
| `-t ed25519` | el tipo de clave. Es el recomendado hoy |
| `-C "curso-rhel"` | una **etiqueta**, no el nombre del archivo. Queda escrita adentro para saber de quién es la clave |

El archivo se llama siempre `id_ed25519`. Conviene dejarlo así: `ssh` lo busca con ese nombre automáticamente.

---

## Paso 3 — Mirar los dos archivos que aparecieron

```powershell
dir $env:USERPROFILE\.ssh
```

**Tiene que haber dos archivos nuevos:**

| Archivo | Qué es |
|---|---|
| `id_ed25519` | la **privada**. Nunca se comparte, con nadie |
| `id_ed25519.pub` | la **pública**. Se copia al servidor |

Veamos qué tiene adentro la pública:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

**Tiene que salir una sola línea** que empieza con `ssh-ed25519` y termina con `curso-rhel`.

Eso es todo lo que es: una línea de texto. Se puede copiar, pegar en un correo, publicar. No sirve para entrar a ningún lado por sí sola.

> **La otra, la que no tiene `.pub`, no se abre y no se manda a nadie.** Si alguna vez alguien les pide ese archivo, les está pidiendo la llave de su casa.

---

## Paso 4 — Copiar la clave pública al servidor

Dos comandos. El primero manda el archivo a la VM:

```powershell
scp -P 2222 $env:USERPROFILE\.ssh\id_ed25519.pub student@localhost:/tmp/miclave.pub
```

Pide la contraseña de `student`.

Ahora entrar a la VM:

```powershell
ssh -p 2222 student@localhost
```

Pide la contraseña otra vez. **Es la última.**

---

## Paso 5 — Instalar la clave, ya dentro de la VM

Estos cuatro comandos se escriben **adentro de la VM** (el prompt dice `[student@rhel01 ~]$`):

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat /tmp/miclave.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

| Comando | Qué hace |
|---|---|
| `mkdir -p ~/.ssh` | crea la carpeta si no existe |
| `chmod 700 ~/.ssh` | solo el dueño entra a la carpeta |
| `cat ... >> authorized_keys` | agrega la línea de la clave al archivo de claves autorizadas |
| `chmod 600 ...` | solo el dueño lee y escribe el archivo |

Los dos `chmod` **no son opcionales**: si los permisos están más abiertos, SSH ignora la clave sin avisar y sigue pidiendo contraseña.

Ahora miremos qué quedó:

```bash
cat ~/.ssh/authorized_keys
```

**Compárenla con la línea del paso 3. Es exactamente la misma.**

No hubo nada mágico: copiamos una línea de texto a un archivo del servidor. Eso es todo lo que hay detrás de "entrar con clave".

Limpiamos el archivo temporal y salimos:

```bash
rm /tmp/miclave.pub
exit
```

---

## Paso 6 — La prueba

De vuelta en PowerShell:

```powershell
ssh -p 2222 student@localhost
```

**Tiene que entrar sin pedir contraseña.**

Si todavía la pide, ir al final de esta hoja, a "Si sigue pidiendo contraseña".

---

## Paso 7 — Dejar de escribir `-p 2222 student@localhost`

Salir de la VM (`exit`) y, en PowerShell:

```powershell
notepad $env:USERPROFILE\.ssh\config
```

Va a decir que el archivo no existe y si quieren crearlo: **Sí**.

Pegar exactamente esto (las cuatro líneas de abajo van indentadas):

```
Host rhel01
    HostName localhost
    Port 2222
    User student
    IdentityFile ~/.ssh/id_ed25519
```

Guardar con *Archivo > Guardar* y cerrar.

> **Ojo:** el archivo tiene que llamarse `config`, **sin** `.txt`. Si Notepad se lo agregó, hay que renombrarlo. Para comprobar: `dir $env:USERPROFILE\.ssh`

Probarlo:

```powershell
ssh rhel01
```

**Tiene que entrar directo.** Sin usuario, sin puerto, sin contraseña.

Y también sirve para ejecutar un comando sin entrar:

```powershell
ssh rhel01 hostname
```

---

## Si sigue pidiendo contraseña

Casi siempre son los permisos. Entrar a la VM con `ssh -p 2222 student@localhost` y comprobar:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

**Tiene que decir:**
```
drwx------.  ... /home/student/.ssh
-rw-------.  ... authorized_keys
```

Si no coincide:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
exit
```

Y volver a intentar.

Si sigue igual, comprobar que la clave llegó de verdad:

```bash
cat ~/.ssh/authorized_keys
```

Tiene que estar la línea que empieza con `ssh-ed25519` y termina con `curso-rhel`.

---

## Dos cosas para tener claras

**La contraseña de `student` sigue existiendo.** No se borró. Se sigue necesitando para `sudo` y para entrar por la ventana del hipervisor. Lo único que cambió es el ingreso por SSH desde esta computadora.

**La clave vive en esta computadora.** Si mañana entran desde otra máquina, esa otra va a pedir la contraseña, porque no tiene la clave privada. Ahí se repiten los pasos 2 a 5 desde la máquina nueva.
