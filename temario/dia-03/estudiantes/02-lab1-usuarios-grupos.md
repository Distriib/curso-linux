# Lab — Crear usuarios y grupos

Vamos a crear cuatro personas y tres áreas. Contraseña de todos: **`Pgn.2026`**.

| Usuario | UID | Descripción | Área |
|---|---|---|---|
| `ana` | 2001 | Ana Rodriguez - Sistemas | `sistemas` (3001) |
| `carlos` | 2002 | Carlos Mendez - Sistemas | `sistemas` (3001) |
| `pedro` | 2003 | Pedro Castillo - Soporte | `soporte` (3002) |
| `laura` | 2004 | Laura Gomez - Auditoria | `auditoria` (3003) |

---

## Parte 1 — Crear los usuarios

**¿Cómo se crea un usuario, con su carpeta, su número y su descripción?**

```bash
sudo useradd -m -u 2001 -c "Ana Rodriguez - Sistemas" ana
```

Ahora ustedes: `carlos`, `pedro` y `laura`, con los datos de la tabla. Foto.

**Comprobar:**
```bash
ls /home
```
```
ana  carlos  laura  pedro  student
```

---

## Parte 2 — Crear los grupos

**¿Cómo se crea un grupo con un número fijo?**

```bash
sudo groupadd -g 3001 sistemas
```

Ahora ustedes: `soporte` (3002) y `auditoria` (3003). Foto.

**Comprobar:**
```bash
getent group sistemas soporte auditoria
```
```
sistemas:x:3001:
soporte:x:3002:
auditoria:x:3003:
```
Todavía sin miembros.

---

## Parte 3 — Meter a cada usuario en su grupo

**¿Cómo agrego a ana al grupo sistemas sin que pierda los grupos que ya tiene?**

```bash
sudo usermod -aG sistemas ana
```

Ahora ustedes: `carlos` a `sistemas`, `pedro` a `soporte`, `laura` a `auditoria`. Foto.

**Comprobar:**
```bash
getent group sistemas soporte auditoria
```
```
sistemas:x:3001:ana,carlos
soporte:x:3002:pedro
auditoria:x:3003:laura
```

---

## Parte 4 — Ver cómo quedaron

```bash
id ana
id carlos
id pedro
id laura
ls -ld /home/*
sudo passwd -S ana
```

**Comprobar:**
```
uid=2001(ana) gid=2001(ana) groups=2001(ana),3001(sistemas)
uid=2002(carlos) gid=2002(carlos) groups=2002(carlos),3001(sistemas)
uid=2003(pedro) gid=2003(pedro) groups=2003(pedro),3002(soporte)
uid=2004(laura) gid=2004(laura) groups=2004(laura),3003(auditoria)
drwx------. 2 ana ana ... /home/ana
...
ana LK ... (Password locked.)
```
`LK`: todavía no tiene contraseña. No puede entrar.

---

## Parte 5 — Contraseñas

**¿Cómo le pongo contraseña a ana?**

```bash
sudo passwd ana
```
(`Pgn.2026` dos veces)

Para no escribirla tres veces más, en una línea:
```bash
echo 'Pgn.2026' | sudo passwd --stdin carlos
```

Ahora ustedes: `pedro` y `laura`. Foto.

**Comprobar:**
```bash
sudo passwd -S ana
sudo passwd -S laura
```
```
ana PS ... (Password set, SHA512 crypt.)
laura PS ... (Password set, SHA512 crypt.)
```

---

## Parte 6 — Cambiar de usuario

**¿Cómo entro como ana para ver que funciona?**

```bash
su - ana
```
(pide la contraseña **de ana**)

Adentro:
```bash
id
pwd
touch mi-archivo.txt
ls -l
exit
```

**Comprobar:** el prompt cambia a `[ana@rhel01 ~]$`. `pwd` da `/home/ana`. El archivo es `-rw-rw-r--. 1 ana ana`. Después del `exit`, el prompt vuelve a `student`.

Ahora ustedes: entren como `pedro`, creen un archivo, salgan. Foto.
