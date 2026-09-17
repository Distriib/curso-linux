# Lab — Auditoría: quién tocó qué y quién intentó entrar (comandos)

Guiado: todos tipean, se lee la salida juntos.

## Parte 1
```bash
systemctl is-active auditd
sudo vim /etc/audit/rules.d/pgn.rules
```
```
## Reglas PGN: cambios en cuentas y en sudoers
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k passwd_changes
-w /etc/group -p wa -k passwd_changes
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers
```
```bash
sudo augenrules --load
sudo auditctl -l
```
Si `auditctl -l` no muestra las reglas: el archivo no termina en `.rules` o quedó en `/etc/audit/` en vez de `/etc/audit/rules.d/`. `sudo augenrules --check` lo dice. Si `augenrules --load` responde `No change`, las reglas ya estaban cargadas: bien.

## Parte 2
```bash
sudo useradd -c "Prueba auditoria" prueba_audit
echo 'Pgn.2026' | sudo passwd --stdin prueba_audit
sudo ausearch -k passwd_changes -ts recent -i | grep type=SYSCALL | tail -3
sudo ausearch -m USER_CMD -ts recent -i | tail -2
```
Las líneas son largas: que busquen `auid=`, `comm=` y `key=` con la vista. **La frase:** "`auid=student`: hicieron el cambio como root, pero la auditoría sabe que entró `student`."

## Parte 3
```bash
sudo aureport --summary | head -14
```

## Parte 4
```bash
ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no admin@localhost true
ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no oracle@localhost true
ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no test@localhost true
sudo grep "Invalid user" /var/log/secure | tail -3
sudo grep -c "Invalid user" /var/log/secure
sudo lastb | head -4
```
`Permission denied (publickey,...)` en cada `ssh` es lo esperado: el Día 8 dejó `PasswordAuthentication no`, así que ni siquiera pide contraseña. `-4` fuerza IPv4: sin él la línea diría `from ::1`. Ellos: un cuarto `ssh` con otro nombre y otra vez el `grep -c`.

## Parte 5
```bash
sudo userdel -r prueba_audit
sudo ausearch -k passwd_changes -ts recent -i | grep -c comm=userdel
```
El aviso `mail spool ... not found` de `userdel` no es un error.
