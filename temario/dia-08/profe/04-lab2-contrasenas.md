# Lab — Contraseñas fuertes y bloqueo por intentos (comandos)

Vos primero, ellos después, foto. Pedir que **escriban en el chat las contraseñas de prueba antes de tipearlas**, para que nadie invente una que sí pase y se pierda el efecto.

## Parte 1 — El usuario de pruebas
```bash
sudo useradd prueba
echo 'Pgn.2026' | sudo passwd --stdin prueba
```
Se crea **antes** de la política para que root le ponga una contraseña de 8 caracteres sin pelear.

## Parte 2 — La política de calidad
```bash
echo "minlen = 12" | sudo tee -a /etc/security/pwquality.conf
echo "minclass = 3" | sudo tee -a /etc/security/pwquality.conf
echo "ocredit = -1" | sudo tee -a /etc/security/pwquality.conf
tail -3 /etc/security/pwquality.conf
```
`tee -a` (Día 2) agrega al final. El archivo de fábrica viene todo comentado, así que las tres últimas líneas son las nuestras.

## Parte 3 — Probarla como el usuario
```bash
su - prueba
passwd
```
Actual `Pgn.2026`; nuevas: `hola123`, `PanamaTech2026`, `abc`. Después `exit`.
pwquality revisa primero el largo: por eso la segunda tiene 14 caracteres, para que el error sea el del símbolo. A los tres intentos `passwd` se cancela y la contraseña sigue siendo `Pgn.2026`.
Si alguien se queda como `prueba` sin darse cuenta (el prompt dice `prueba@`), todo lo que siga con `sudo` falla: que haga `exit`.

## Parte 4 — Activar el bloqueo por intentos
```bash
sudo authselect current
sudo authselect enable-feature with-faillock
sudo authselect current
grep faillock /etc/pam.d/system-auth
```
El perfil puede ser `sssd` o `local`, y `Enabled features` puede traer algo ya prendido: no importa. Si `authselect current` responde `No existing configuration detected`: `sudo authselect select sssd --force` y repetir el `enable-feature`. Si se queja de `Unexpected changes to the PAM configuration`: `sudo authselect select sssd with-faillock --force`.

## Parte 5 — Los umbrales
```bash
echo "deny = 5" | sudo tee -a /etc/security/faillock.conf
echo "unlock_time = 900" | sudo tee -a /etc/security/faillock.conf
tail -2 /etc/security/faillock.conf
```
Los valores de fábrica (comentados) son 3 intentos y 600 segundos. Los nuestros van al final y valen.

## Parte 6 — Cinco intentos fallidos
Cinco veces, con una contraseña incorrecta (`mala`):
```bash
su - prueba
```
Después:
```bash
sudo faillock --user prueba
```
Si alguien tipea la correcta por error y entra: `exit`, y ese intento no cuenta; que siga hasta juntar cinco fallidos. La columna `Valid` con `V` = el intento cuenta.

## Parte 7 — Con la contraseña correcta, sigue bloqueado
```bash
su - prueba
```
(`Pgn.2026`)
```bash
sudo journalctl --since "5 min ago" | grep locked
```
Qué decir: "la contraseña es correcta y no entra. Eso es el bloqueo. Dura 15 minutos, o hasta que un administrador lo levante."

## Parte 8 — Desbloquear
```bash
sudo faillock --user prueba --reset
sudo faillock --user prueba
su - prueba
```
(`Pgn.2026`; adentro `exit`)
faillock no bloquea a root por defecto: bloquear a root remoto es deseable, pero solo si hay consola. Solo si alguien lo pregunta.
