# 1 — Quién es quién

## Para el sistema no hay nombres, hay números

```bash
id
id root
```

- **UID** = número del usuario. **GID** = número del grupo.
- Los nombres (`student`, `root`, `ana`) son traducciones que se hacen mirando `/etc/passwd`.
- **root es UID 0.** No es especial por el nombre: es especial porque al UID 0 el sistema no le aplica los permisos.

| UID | Quién |
|---|---|
| `0` | root |
| `1`–`999` | cuentas de sistema (servicios: `sshd`, `chrony`…) |
| `1000`+ | usuarios normales (`student` es 1000: fue el primero) |
| `65534` | `nobody` |

## `/etc/passwd` — la lista de cuentas (7 campos)

```bash
getent passwd student root
```

```
student:x:1000:1000:student:/home/student:/bin/bash
   1    2   3    4     5          6            7
```
1 nombre · 2 `x` (la contraseña está en `shadow`) · 3 UID · 4 GID del grupo **primario** · 5 descripción · 6 home · 7 shell (`/sbin/nologin` = no puede iniciar sesión)

## `/etc/shadow` — las contraseñas (9 campos, solo root)

```bash
sudo grep student /etc/shadow
```

```
student:$6$Zy9k...:20699:0:99999:7:::
   1        2        3   4   5   6 7 8 9
```
1 nombre · 2 contraseña cifrada · 3 último cambio (días desde 1970) · 4 mínimo entre cambios · 5 máximo de validez · 6 aviso antes de expirar · 7 inactividad · 8 expiración de la **cuenta** · 9 reservado

Campo 2:

| Empieza con | Significa |
|---|---|
| `$6$` | contraseña cifrada (SHA-512) |
| `!!` | nunca se le puso contraseña |
| `!` o `!!` delante de `$6$` | contraseña **bloqueada** |
| `*` | cuenta de sistema, nunca tendrá contraseña |

## `/etc/group` — los grupos

```bash
getent group wheel student
```
getent = get entries: "traeme la entrada de tal cosa en tal base de datos". 
getent group wheel student = "de la base de grupos, mostrame wheel y student".

```
wheel:x:10:student
  1   2  3    4
```
1 nombre · 2 `x` · 3 GID · 4 miembros **suplementarios**

- **Grupo primario:** el del campo 4 de `passwd`. Con ese grupo nacen los archivos que el usuario crea.
- **Grupos suplementarios:** los de `/etc/group`. Dan acceso adicional.
- RHEL crea a cada usuario un **grupo privado** con su mismo nombre (`student` → grupo `student`).

## Los cuatro archivos y sus permisos

```bash
ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow
```

## Quién está y quién estuvo

| Comando | Muestra |
|---|---|
| `whoami` | yo |
| `id` | mis UID, GID y grupos |
| `who` / `w` | quién está conectado ahora (`w` agrega qué está haciendo) |
| `last` | historial de sesiones |
| `sudo lastb` | intentos fallidos |
| `lastlog` | último ingreso de cada cuenta |

## Permisos en carpetas

```
-rw-rw-r--. 1 student student ... /home/student/prueba.txt
   │  │  │      │       └ grupo dueño: student
   │  │  │      └ dueño: student
   │  │  └ otros: cualquiera que no sea student → ana, pedro, root...
   │  └ grupo: los del grupo student (solo vos)
   └ dueño: vos
```