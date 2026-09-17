# Lab 2.4 — Auditoría: quién tocó qué y quién intentó entrar

Todos corren cada comando; después lo leemos juntos. Objetivo: vigilar `/etc/passwd`, `/etc/shadow` y `sudoers` con auditd, encontrar quién los tocó, y leer los intentos de acceso fallidos por SSH.

---

## Parte 1 — La regla, persistente

**¿Cómo hago que quede registrado cualquier cambio en las cuentas?**

```bash
systemctl is-active auditd
sudo vim /etc/audit/rules.d/pgn.rules
```
Contenido:
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

**Comprobar:**
```
active
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k passwd_changes
-w /etc/group -p wa -k passwd_changes
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d -p wa -k sudoers
```
`augenrules --load` no imprime nada cuando carga bien. Las reglas sobreviven al reinicio porque viven en `rules.d/`.

---

## Parte 2 — Provocar y buscar por llave

**¿Quién creó una cuenta, y con qué comando?**

```bash
sudo useradd -c "Prueba auditoria" prueba_audit
echo 'Pgn.2026' | sudo passwd --stdin prueba_audit
sudo ausearch -k passwd_changes -ts recent -i | grep type=SYSCALL | tail -3
sudo ausearch -m USER_CMD -ts recent -i | tail -2
```

**Comprobar** (líneas largas; lo que importa):
```
type=SYSCALL ... auid=student uid=root ... comm=useradd exe=/usr/sbin/useradd ... key=passwd_changes
type=SYSCALL ... auid=student uid=root ... comm=passwd exe=/usr/bin/passwd ... key=passwd_changes
type=USER_CMD ... auid=student ... cmd=useradd -c "Prueba auditoria" prueba_audit ... res=success
```
`auid` es quién **inició la sesión**: el comando corrió como root por `sudo`, pero la auditoría dice `student`. `USER_CMD` son los comandos ejecutados con `sudo`.

---

## Parte 3 — El resumen

```bash
sudo aureport --summary | head -14
```

**Comprobar:**
```
Summary Report
======================
Range of time in logs: ...
Number of changes in configuration: ...
Number of changes to accounts, groups, or roles: ...
Number of logins: ...
Number of failed logins: ...
Number of authentications: ...
Number of failed authentications: ...
```
Es el informe que se adjunta a un incidente.

---

## Parte 4 — Quién intentó entrar por SSH

**¿Cómo veo los intentos de acceso con usuarios que no existen?**

```bash
ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no admin@localhost true
ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no oracle@localhost true
ssh -4 -o BatchMode=yes -o StrictHostKeyChecking=no test@localhost true
sudo grep "Invalid user" /var/log/secure | tail -3
sudo grep -c "Invalid user" /var/log/secure
sudo lastb | head -4
```

Ahora ustedes: prueben con un cuarto usuario inventado y vuelvan a contar. Foto.

**Comprobar:**
```
admin@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
oracle@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
test@localhost: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
Sep 16 ... rhel01 sshd[NNNN]: Invalid user admin from 127.0.0.1 port NNNNN
Sep 16 ... rhel01 sshd[NNNN]: Invalid user oracle from 127.0.0.1 port NNNNN
Sep 16 ... rhel01 sshd[NNNN]: Invalid user test from 127.0.0.1 port NNNNN
3
btmp begins ...
```
Con esta lista se decide qué IP bloquear en el firewall. `lastb` puede salir vacío: los intentos por clave no pasan por donde él mira; `secure` sí los tiene.

---

## Parte 5 — Limpiar (la regla se queda)

```bash
sudo userdel -r prueba_audit
sudo ausearch -k passwd_changes -ts recent -i | grep -c comm=userdel
```

**Comprobar:** un número mayor que `0`. `userdel` también tocó `passwd`, `shadow` y `group`, y quedó registrado.
