# Lab — Entrar sin contraseña (guía del instructor)

Lab autocontenido: solo necesita la VM del Día 1 con el reenvío `2222→22`. Los participantes están en **Windows**; los comandos del archivo de estudiantes son todos de PowerShell.

---

## Qué es cada cosa

**Qué es un par de claves:** dos archivos que se generan juntos. Lo que cierra uno solo lo abre el otro. La **privada** se queda en la computadora del participante; la **pública** se copia al servidor.
**Cómo funciona el ingreso:** el servidor guarda la clave pública. Al conectarse, le manda al cliente un desafío que **solo puede resolver quien tenga la privada**. Si lo resuelve, entra. La contraseña no viaja, y de hecho ya no se usa.
**Qué es `authorized_keys`:** el archivo del servidor con la lista de claves públicas autorizadas para ese usuario. Una clave por línea.
**Qué es `ed25519`:** el tipo de clave recomendado hoy. Antes se usaba `rsa` de 4096 bits, que sigue siendo válido en sistemas viejos.
**Qué es `-C "curso-rhel"`:** un **comentario**, no el nombre del archivo. Queda escrito al final de la línea de la clave pública, para saber de quién es cuando un servidor tiene diez claves autorizadas.
**Por qué el nombre por defecto (`id_ed25519`):** porque `ssh` lo busca con ese nombre automáticamente. Con otro nombre habría que indicárselo en cada conexión (`-i`).
**Qué es la passphrase:** una contraseña que protege el archivo de la clave privada, por si alguien se lo roba. En el curso se deja vacía.
**Qué es `scp`:** copiar archivos por SSH. `-P` mayúscula para el puerto (en `ssh` es `-p` minúscula).

---

## Por qué este lab usa `scp` y no `ssh-copy-id`

`ssh-copy-id` **no existe en Windows**. La alternativa que documenta Microsoft es pasar el archivo por una tubería:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 student@localhost "cat >> ~/.ssh/authorized_keys"
```

Funciona casi siempre, pero PowerShell a veces altera el texto al pasarlo por la tubería y la clave llega corrupta — y el síntoma es el mismo que el de los permisos mal puestos: sigue pidiendo contraseña, sin decir por qué.

Por eso el lab usa `scp` + `cat` adentro de la VM:

- **Nunca falla por codificación.**
- Ellos **ven viajar el archivo** y **lo agregan con sus manos** al `authorized_keys`. Pedagógicamente es mejor: queda clarísimo que no hay magia, que es copiar una línea de texto.
- De paso aprenden `scp`, que se usa igual.

---

## Cómo darlo

**La demo conviene hacerla dentro de la VM**, contra sí misma: ahí los comandos son iguales para todos y no tenés que mostrar Mac mientras ellos escriben Windows.

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/demo -C "demo-clase"
ls -l ~/.ssh
cat ~/.ssh/demo.pub
cat ~/.ssh/demo >/dev/null; head -2 ~/.ssh/demo
```

Mostrás los dos archivos, abrís el `.pub` (una línea de texto), señalás que el otro empieza con `BEGIN OPENSSH PRIVATE KEY`, y explicás el dibujo de la hoja de ellos. Después borrás la demo (`rm ~/.ssh/demo*`) y que ellos hagan el lab en su Windows.

---

## Comandos del lab, paso por paso

### Paso 0 — en PowerShell
```powershell
ssh -V
```
Si falta: *Configuración > Aplicaciones > Características opcionales > Agregar > Cliente de OpenSSH*.

### Paso 1
```powershell
dir $env:USERPROFILE\.ssh
```

### Paso 2
```powershell
ssh-keygen -t ed25519 -C "curso-rhel"
```
Enter, Enter, Enter.

### Paso 3
```powershell
dir $env:USERPROFILE\.ssh
type $env:USERPROFILE\.ssh\id_ed25519.pub
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

### Paso 4
```powershell
scp -P 2222 $env:USERPROFILE\.ssh\id_ed25519.pub student@localhost:/tmp/miclave.pub
ssh -p 2222 student@localhost
```

### Paso 5 — dentro de la VM
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat /tmp/miclave.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
rm /tmp/miclave.pub
exit
```

### Paso 6
```powershell
ssh -p 2222 student@localhost
```

### Paso 7
```powershell
notepad $env:USERPROFILE\.ssh\config
```
Contenido:
```
Host rhel01
    HostName localhost
    Port 2222
    User student
    IdentityFile ~/.ssh/id_ed25519
```
```powershell
ssh rhel01
ssh rhel01 hostname
```

---

## Qué señalar en cada paso

**Paso 1** — *"miren que no hay nada todavía"*. Sirve para que el contraste del paso 3 se note.

**Paso 2** — aclarar que el `-C` es una **etiqueta**, no el nombre. Va a salir la pregunta.

**Paso 3 — el primero de los dos momentos importantes.** Tres cosas:
- Aparecieron **dos** archivos de un solo comando.
- El `.pub` es **una línea de texto legible**: *"esto no es un archivo secreto, se puede pegar en un correo"*.
- El otro, no. *"Si alguna vez alguien les pide el archivo sin `.pub`, les está pidiendo la llave de su casa. No hay ningún motivo legítimo para mandarla."*

**Paso 4** — la contraseña se escribe **por última vez**. Decirlo.

**Paso 5 — el segundo momento, y el que hace entender todo.** Después del `cat ~/.ssh/authorized_keys`, que **comparen esa línea con la del paso 3**. Es idéntica.
Frase: *"eso es todo lo que hay detrás de 'entrar con clave': una línea de texto copiada a un archivo del servidor. Podrían haberla pegado a mano."*
Los dos `chmod` no son decorativos: si quedan mal, SSH ignora la clave **en silencio**.

**Paso 6** — entra sin preguntar nada.

**Paso 7** — el premio práctico. En un puesto de administración real ese archivo tiene decenas de servidores con su puerto, su usuario y su clave.

---

## Si a alguien le sigue pidiendo contraseña

En este orden, adentro de la VM:

1. `ls -ld ~/.ssh` → tiene que ser `drwx------`
2. `ls -l ~/.ssh/authorized_keys` → tiene que ser `-rw-------`
3. `cat ~/.ssh/authorized_keys` → tiene que estar la línea completa, empezando con `ssh-ed25519`
4. Si todo lo anterior está bien, desde PowerShell mirar qué pasa:
   `ssh -v -p 2222 student@localhost exit`
   Buscar las líneas `Offering public key` y `Server accepts key`.

---

## Errores que vas a ver

| Pasa | Por qué | Solución |
|---|---|---|
| Sigue pidiendo contraseña | Permisos de `~/.ssh` o de `authorized_keys`. **Es casi siempre esto** | `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys` |
| `ssh: command not found` en PowerShell | Falta el cliente OpenSSH | Características opcionales de Windows |
| `scp` da error de sintaxis | Usaron `-p` minúscula. En `scp` el puerto es `-P` | `-P 2222` |
| El archivo quedó como `config.txt` | Notepad agregó la extensión | Renombrarlo a `config` |
| `Bad owner or permissions on config` | Raro en Windows, pasa si el archivo quedó en otra carpeta | Comprobar que esté en `.ssh` |
| El alias no funciona | Las líneas después de `Host` no están indentadas | Corregir la sangría |
| Sobrescribieron una clave que ya tenían | Respondieron `y` al "ya existe" | Rehacer los pasos 4 y 5 con la nueva |
| Pusieron passphrase sin querer | Ahora la pide cada vez | Está bien (es más seguro), o rehacer la clave |

---

## Cierre

> "La contraseña de `student` sigue existiendo: la necesitan para `sudo` y para entrar por la ventana del hipervisor. Lo único que cambió es el ingreso por SSH desde esta computadora.
>
> Y esto no es comodidad: así se administra un servidor de verdad. Nadie escribe contraseñas. Es además el primer paso para automatizar, porque un script no puede escribir una contraseña, pero sí puede usar una clave."
