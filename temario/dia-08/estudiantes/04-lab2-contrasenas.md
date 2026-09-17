# Lab 4.2 — Contraseñas fuertes y bloqueo por intentos

Vamos a exigir contraseñas de 12 caracteres con tres clases y un símbolo, bloquear la cuenta tras cinco intentos fallidos, y desbloquearla.

| Dato | Valor |
|---|---|
| Usuario de pruebas | `prueba`, contraseña `Pgn.2026` |
| Política | `minlen = 12`, `minclass = 3`, `ocredit = -1` |
| Bloqueo | `deny = 5` intentos, `unlock_time = 900` segundos |

---

## Parte 1 — El usuario de pruebas

**¿Por qué lo creamos antes de endurecer la política?**

```bash
sudo useradd prueba
echo 'Pgn.2026' | sudo passwd --stdin prueba
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** `passwd: all authentication tokens updated successfully.` Root le puso una contraseña corta sin problema: la política se le aplica al usuario, no al administrador.

---

## Parte 2 — La política de calidad

**¿Dónde se escriben las reglas de la contraseña?**

```bash
echo "minlen = 12" | sudo tee -a /etc/security/pwquality.conf
echo "minclass = 3" | sudo tee -a /etc/security/pwquality.conf
echo "ocredit = -1" | sudo tee -a /etc/security/pwquality.conf
tail -3 /etc/security/pwquality.conf
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
minlen = 12
minclass = 3
ocredit = -1
```
Aplica de inmediato: se lee en cada `passwd`.

---

## Parte 3 — Probarla como el usuario

**¿Qué le pasa a alguien que quiere poner una contraseña débil?**

```bash
su - prueba
passwd
```
Contraseña actual: `Pgn.2026`. Como nueva, en este orden: `hola123`, después `PanamaTech2026`, después `abc`. Al final, `exit`.

Ahora ustedes: lo mismo, con esas tres. Foto.

**Comprobar:**
```
Changing password for user prueba.
Current password:
New password:
BAD PASSWORD: The password is shorter than 12 characters
New password:
BAD PASSWORD: The password contains less than 1 non-alphanumeric characters
New password:
BAD PASSWORD: The password is shorter than 12 characters
passwd: Have exhausted maximum number of retries for service
```
La segunda tiene 14 caracteres y tres clases, pero ningún símbolo. La contraseña sigue siendo `Pgn.2026`.

---

## Parte 4 — Activar el bloqueo por intentos

**¿Cómo se activa faillock sin editar PAM a mano?**

```bash
sudo authselect current
sudo authselect enable-feature with-faillock
sudo authselect current
grep faillock /etc/pam.d/system-auth
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Profile ID: sssd
Enabled features: None
Profile ID: sssd
Enabled features:
- with-faillock
auth        required                                     pam_faillock.so preauth silent
auth        required                                     pam_faillock.so authfail
account     required                                     pam_faillock.so
```
El perfil puede llamarse distinto; lo que importa es que después aparezca `with-faillock`, y que PAM tenga las tres líneas.

---

## Parte 5 — Los umbrales

**¿Cuántos intentos y por cuánto tiempo?**

```bash
echo "deny = 5" | sudo tee -a /etc/security/faillock.conf
echo "unlock_time = 900" | sudo tee -a /etc/security/faillock.conf
tail -2 /etc/security/faillock.conf
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
deny = 5
unlock_time = 900
```

---

## Parte 6 — Cinco intentos fallidos

**¿Qué queda registrado?**

Cinco veces, escribiendo una contraseña **incorrecta** a propósito (por ejemplo `mala`):
```bash
su - prueba
```
Después:
```bash
sudo faillock --user prueba
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
su: Authentication failure
(cinco veces)
prueba:
When                Type  Source                                           Valid
2026-09-16 11:32:10 TTY   pts/0                                                V
... (cinco filas)
```

---

## Parte 7 — Con la contraseña correcta, sigue bloqueado

**¿Y ahora con la buena?**

```bash
su - prueba
```
(escribir `Pgn.2026`, la correcta)
```bash
sudo journalctl --since "5 min ago" | grep locked
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:** `su: Authentication failure` aunque la contraseña sea correcta, y en el journal una línea con `pam_faillock(su-l:auth): Consecutive login failures for user prueba account temporarily locked`.

---

## Parte 8 — Desbloquear

**¿Cómo lo desbloqueo antes de los 15 minutos?**

```bash
sudo faillock --user prueba --reset
sudo faillock --user prueba
su - prueba
```
(escribir `Pgn.2026`; adentro, `exit`)

Ahora ustedes: lo mismo. Foto.

**Comprobar:** la lista queda vacía (solo la línea `prueba:`), y `su` entra: el prompt cambia a `prueba`. `exit` vuelve a `student`.
