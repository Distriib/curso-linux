# Lab — Crear usuarios y grupos (comandos)

Contraseña de todos: `Pgn.2026`. Yo hago el primero, ellos los demás, foto.

## Parte 1 — Crear los usuarios
```bash
sudo useradd -m -u 2001 -c "Ana Rodriguez - Sistemas" ana
sudo useradd -m -u 2002 -c "Carlos Mendez - Sistemas" carlos
sudo useradd -m -u 2003 -c "Pedro Castillo - Soporte" pedro
sudo useradd -m -u 2004 -c "Laura Gomez - Auditoria" laura
ls /home
```

## Parte 2 — Crear los grupos
```bash
sudo groupadd -g 3001 sistemas
sudo groupadd -g 3002 soporte
sudo groupadd -g 3003 auditoria
getent group sistemas soporte auditoria
```

## Parte 3 — Meter a cada usuario en su grupo
```bash
sudo usermod -aG sistemas ana
sudo usermod -aG sistemas carlos
sudo usermod -aG soporte pedro
sudo usermod -aG auditoria laura
getent group sistemas soporte auditoria
```
Si alguien lo hace sin `-a`, no pasa nada grave hoy (los usuarios nuevos no estaban en ningún grupo). Decirlo igual: "con un usuario que ya está en `wheel`, sin `-a` lo sacás de `wheel`".

## Parte 4 — Ver cómo quedaron
```bash
id ana
id carlos
id pedro
id laura
ls -ld /home/*
sudo passwd -S ana
```

## Parte 5 — Contraseñas
```bash
sudo passwd ana
```
(`Pgn.2026` dos veces)
```bash
echo 'Pgn.2026' | sudo passwd --stdin carlos
echo 'Pgn.2026' | sudo passwd --stdin pedro
echo 'Pgn.2026' | sudo passwd --stdin laura
sudo passwd -S ana
sudo passwd -S laura
```

## Parte 6 — Cambiar de usuario
```bash
su - ana
```
Como ana:
```bash
id
pwd
touch mi-archivo.txt
ls -l
exit
```
Ellos lo repiten con `pedro`.

Si alguien se queda como `ana` sin darse cuenta (el prompt dice `ana@`), todo lo que haga después falla con `Permission denied`. Que mire el prompt y haga `exit`.
