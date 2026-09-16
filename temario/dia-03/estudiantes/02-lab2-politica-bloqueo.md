# Lab — Política de contraseñas y bloqueo

Política: la contraseña dura **90 días**, aviso **7 días** antes, mínimo **1 día** entre cambios.

---

## Parte 1 — Cómo está ana ahora

```bash
sudo chage -l ana
```

**Comprobar:** `Password expires: never`, `Maximum number of days: 99999`. Sin política.

---

## Parte 2 — Aplicar la política

**¿Cómo le pongo a ana los tres plazos?**

```bash
sudo chage -M 90 -m 1 -W 7 ana
sudo chage -l ana
```

Ahora ustedes: `carlos`, `pedro` y `laura`. Foto.

**Comprobar:** `Maximum: 90`, `Minimum: 1`, `warning: 7`, y `Password expires` con una fecha 90 días adelante.

---

## Parte 3 — Que los usuarios nuevos nazcan así

```bash
sudo vim /etc/login.defs
```
Buscar `PASS_MAX_DAYS` (`/PASS_MAX_DAYS`), cambiar `99999` por `90`, guardar (`:wq`).

```bash
grep PASS_MAX_DAYS /etc/login.defs
```

**Comprobar:** la línea que **no** empieza con `#` dice `PASS_MAX_DAYS	90`.

---

## Parte 4 — Obligar a cambiar la contraseña al entrar

```bash
sudo chage -d 0 laura
su - laura
```
Escribir `Pgn.2026`, leer el mensaje, y **cancelar con `Ctrl+C`**.

```bash
sudo chage -d "$(date +%F)" laura
```

**Comprobar:** el mensaje es `You are required to change your password immediately`. El último comando vuelve a dejar la fecha de hoy.

---

## Parte 5 — Bloquear la contraseña

```bash
sudo passwd -l pedro
sudo passwd -S pedro
su - pedro
```
(escribir `Pgn.2026`)

```bash
sudo passwd -u pedro
sudo passwd -S pedro
```

**Comprobar:** `pedro LK`, el `su` da `Authentication failure`. Después de `-u`, `pedro PS`.

---

## Parte 6 — Una cuenta temporal, sin shell

```bash
sudo useradd -m -u 2099 -c "Practicante temporal" temporal
echo 'Pgn.2026' | sudo passwd --stdin temporal
sudo usermod -s /sbin/nologin temporal
su - temporal
```
(escribir `Pgn.2026`)

**Comprobar:** `This account is currently not available.` La contraseña fue correcta; la shell no deja entrar.

---

## Parte 7 — Expirar la cuenta

```bash
sudo chage -E 0 temporal
su - temporal
```

**Comprobar:** `Your account has expired; please contact your system administrator.`

---

## Parte 8 — Borrar la cuenta

```bash
sudo userdel -r temporal
getent passwd temporal
ls /home
```

**Comprobar:** `getent` no devuelve nada. `temporal` ya no está en `/home`.
