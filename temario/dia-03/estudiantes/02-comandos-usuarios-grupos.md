# Comandos — usuarios y grupos

## Crear un usuario

```bash
sudo useradd -m -u 2001 -c "Ana Rodriguez - Sistemas" ana
```

| Parte | Qué hace |
|---|---|
| `-m` | crea su carpeta `/home/ana` |
| `-u 2001` | le pone el número 2001 (sin `-u`, el sistema elige) |
| `-c "..."` | descripción: nombre y área |
| `ana` | el nombre de usuario |

Un `useradd` hace cinco cosas solo: una línea en `/etc/passwd`, una en `/etc/shadow` (sin contraseña, `!!`), un grupo privado `ana`, la carpeta `/home/ana` con permisos `700`, y un buzón de correo.

## Crear un grupo

```bash
sudo groupadd -g 3001 sistemas
```

`-g 3001` = el número del grupo (sin `-g`, el sistema elige).

## Meter un usuario en un grupo

```bash
sudo usermod -aG sistemas ana
```

| Parte | Qué hace |
|---|---|
| `-a` | **agregar** (sin `-a`, reemplaza todos sus grupos — ⚠️ los pierde) |
| `-G sistemas` | al grupo `sistemas` |

Alternativa que solo agrega, sin riesgo: `sudo gpasswd -a ana sistemas`. Para sacar: `sudo gpasswd -d ana sistemas`.

## Ver cómo quedó

```bash
id ana
getent group sistemas
ls -ld /home/ana
sudo passwd -S ana
```

`passwd -S` dice: `LK` = sin contraseña o bloqueada · `PS` = tiene contraseña · `NP` = sin contraseña.

## Ponerle contraseña

```bash
sudo passwd ana
```
Pide la contraseña dos veces. Root puede poner una débil (avisa, pero acepta).

En una sola línea, sin que pregunte:
```bash
echo 'Pgn.2026' | sudo passwd --stdin carlos
```

## Cambiar de usuario

```bash
su - ana
```
Pide la contraseña **de ana**. El prompt cambia a `[ana@rhel01 ~]$`. `exit` vuelve.

---

## Plazos y bloqueo — para el Lab 2, si hay tiempo

| Comando | Qué hace |
|---|---|
| `sudo chage -l ana` | ver los plazos de su contraseña |
| `sudo chage -M 90 -m 1 -W 7 ana` | dura 90 días, mínimo 1 entre cambios, aviso 7 días antes |
| `sudo chage -d 0 ana` | obligarla a cambiarla en el próximo ingreso |
| `sudo passwd -l ana` / `-u ana` | bloquear / desbloquear la contraseña |
| `sudo usermod -s /sbin/nologin ana` | quitarle la shell (no puede iniciar sesión) |
| `sudo chage -E 0 ana` | expirar la **cuenta** completa |
| `sudo userdel -r ana` | borrar cuenta, carpeta y buzón |
