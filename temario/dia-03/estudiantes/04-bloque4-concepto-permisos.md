# 4 — Permisos, `umask`, bits especiales y ACL

## Leer `ls -l`, ahora completo

```bash
ls -l /etc/hostname /usr/bin/passwd
ls -ld /tmp ~
```

```
-rwxr-x---. 1 ana sistemas 25 Sep 17 11:00 plan.txt
│└┬┘└┬┘└┬┘│
│ u   g   o └ . = SELinux    + = tiene ACL
│ │   │   └ otros
│ │   └ grupo dueño
│ └ usuario dueño
└ tipo
```

El sistema mira **una sola** terna: si sos el dueño, `u` y nada más; si no, y estás en el grupo, `g`; si no, `o`.

## Octal

`r=4  w=2  x=1`, se suma por terna.

| Octal | Letras | Uso |
|---|---|---|
| `755` | `rwxr-xr-x` | programas, carpetas públicas |
| `644` | `rw-r--r--` | archivos normales |
| `700` | `rwx------` | tu home |
| `600` | `rw-------` | secretos |
| `750` | `rwxr-x---` | carpeta de área, el grupo solo lee |
| `2770` | `rwxrws---` | carpeta de área **con herencia de grupo** |
| `1777` | `rwxrwxrwt` | buzón público (`/tmp`) |

## `chmod`

| Forma | Ejemplo |
|---|---|
| Octal | `chmod 640 archivo` |
| Simbólico | `chmod u+x archivo` · `chmod g-w archivo` · `chmod o= archivo` · `chmod u=rw,g=r,o= archivo` |
| Recursivo | `chmod -R g+rX carpeta` (la `X` mayúscula pone `x` solo a carpetas) |

## Permisos en una carpeta

| Bit | En una carpeta significa |
|---|---|
| `r` | ver los nombres |
| `x` | entrar y usar lo que hay adentro |
| `w` (+ `x`) | crear, renombrar y **borrar** adentro |

Borrar un archivo depende de los permisos de la **carpeta**, no del archivo.

## `chown` y `chgrp`

```bash
sudo chown ana archivo
sudo chown ana:sistemas archivo
chgrp sistemas archivo
sudo chown -R root:sistemas carpeta
```
Solo root cambia el dueño. El dueño puede cambiar el grupo a uno al que pertenece.

## `umask` — con qué permisos nacen las cosas

```bash
umask
umask -S
```

Se **resta** de `666` (archivos) o `777` (carpetas):

| `umask` | Archivos | Carpetas | Quién |
|---|---|---|---|
| `0022` | `644` | `755` | root |
| `0002` | `664` | `775` | usuarios con grupo privado (`student`, `ana`…) |
| `0077` | `600` | `700` | paranoico |

## Bits especiales — el cuarto dígito

| Bit | Octal | Se ve como | En una carpeta | En un programa |
|---|---|---|---|---|
| setuid | `4` | `rws` (en `u`) | nada | corre como el **dueño** del archivo (`/usr/bin/passwd`) |
| setgid | `2` | `rws` (en `g`) | lo nuevo **hereda el grupo** de la carpeta | corre con el grupo del archivo |
| sticky | `1` | `rwt` (en `o`) | solo el dueño borra sus archivos (`/tmp`) | nada |

`S` o `T` mayúscula = el bit está pero falta el `x`: error.

```bash
sudo find / -xdev -type f -perm -4000 2>/dev/null | head
```

## ACL — cuando ugo no alcanza

Para dar permiso a **otro** grupo o a **un** usuario suelto, sin cambiar dueño ni grupo.

| Comando | Qué hace |
|---|---|
| `getfacl carpeta` | ver |
| `sudo setfacl -m g:auditoria:rX carpeta` | dar al grupo `auditoria` lectura (y entrada a carpetas) |
| `sudo setfacl -m u:pedro:r archivo` | dar a pedro lectura |
| `sudo setfacl -R -m ...` | recursivo, a lo que ya existe |
| `sudo setfacl -d -m g:auditoria:rx carpeta` | **por defecto**: lo que se cree adentro la hereda |
| `sudo setfacl -x g:auditoria carpeta` | quitar esa entrada |
| `sudo setfacl -b carpeta` | quitar todas |

En `ls -l`, el `.` se vuelve `+` cuando hay ACL. Una ACL en un archivo no sirve si no podés **entrar** a la carpeta.
