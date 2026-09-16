# Lab — Política de contraseñas y bloqueo (comandos)

## Parte 1
```bash
sudo chage -l ana
```

## Parte 2
```bash
sudo chage -M 90 -m 1 -W 7 ana
sudo chage -M 90 -m 1 -W 7 carlos
sudo chage -M 90 -m 1 -W 7 pedro
sudo chage -M 90 -m 1 -W 7 laura
sudo chage -l ana
```

## Parte 3
```bash
sudo vim /etc/login.defs
```
`/PASS_MAX_DAYS` `Enter` → cambiar `99999` por `90` → `Esc` `:wq`.
```bash
grep PASS_MAX_DAYS /etc/login.defs
```

# Si sobra tiempo

## Parte 4
```bash
sudo chage -d 0 laura
su - laura
```
`Pgn.2026`, leer el mensaje, **`Ctrl+C`**.
```bash
sudo chage -d "$(date +%F)" laura
```
Si alguien completó el cambio y perdió la clave: `echo 'Pgn.2026' | sudo passwd --stdin laura` y otra vez el `chage -d`.

## Parte 5
```bash
sudo passwd -l pedro - bloquea solo password
sudo passwd -S pedro - Muestra el estado
su - pedro
```
(falla)
```bash
sudo passwd -u pedro - quita el bloqueo
sudo passwd -S pedro - mostramos que sirve
```

## Parte 6 — quitar la shell
```bash
sudo useradd -m -u 2099 -c "Practicante temporal" temporal
echo 'Pgn.2026' | sudo passwd --stdin temporal
sudo usermod -s /sbin/nologin temporal - cambiamos la shell por sin shell
su - temporal
```

## Parte 7
```bash
sudo chage -E 0 temporal
su - temporal
```

## Parte 8
```bash
sudo userdel -r temporal
getent passwd temporal
ls /home
```
