# 3 — Privilegios: `su`, `sudo` y `sudoers`

## Tres caminos hacia root

| Camino | Pide | Queda registrado |
|---|---|---|
| Entrar directo como root | la clave de root | "entró root" — no se sabe quién |
| `su -` | la clave de **root** | "alguien hizo su" — no qué hizo después |
| `sudo comando` | **tu** clave | quién, cuándo, qué comando |

`sudo` es el estándar: registra, delega comandos concretos, y se revoca quitando una línea.

## `su` vs `su -`

```bash
su ana -c 'pwd; echo $HOME'
su - ana -c 'pwd; echo $HOME'
```

- `su ana` → cambia de usuario pero **conserva tu entorno** (carpeta actual, variables).
- `su - ana` → abre una sesión completa de ana, como si ella se conectara. **Usar siempre el guion.**

## `sudo` — uso diario

| Comando | Qué hace |
|---|---|
| `sudo comando` | ejecutar como root, con tu contraseña |
| `sudo -l` | ¿qué puedo ejecutar? |
| `sudo -l -U pedro` | ¿qué puede ejecutar pedro? |
| `sudo -i` | shell de root, con tu contraseña |
| `sudo -u ana comando` | ejecutar como ana |
| `sudo -k` | olvidar la contraseña guardada (dura 5 min) |

## `/etc/sudoers` — quién puede qué

```bash
sudo grep wheel /etc/sudoers
sudo grep includedir /etc/sudoers
ls -l /etc/sudoers /etc/sudoers.d
```

- Se edita **solo con `visudo`**: valida antes de guardar. Un `sudoers` roto deja a **todos** sin `sudo`.
- Las reglas propias van en **`/etc/sudoers.d/`**, un archivo por delegación: `sudo visudo -f /etc/sudoers.d/nombre`.
- `%wheel ALL=(ALL) ALL` → todo miembro de `wheel` puede todo. `student` está en `wheel`.

## Anatomía de una regla

```
%soporte   ALL=(root)   /usr/bin/systemctl restart chronyd
   │        │    │              └ qué: ruta completa y argumentos exactos
   │        │    └ como quién
   │        └ en qué servidor (ALL = cualquiera)
   └ quién: usuario, o %grupo
```

- Con argumentos escritos, tienen que coincidir **exactamente**. Sin argumentos, vale cualquiera.
- `NOPASSWD:` delante del comando = sin contraseña. Solo para automatización. **Nunca `NOPASSWD: ALL`.**
- ⚠️ Nunca delegar `vim`, `less`, `bash`, `find`: desde adentro se abre una shell de root.

## Auditoría

```bash
sudo grep sudo /var/log/secure | tail -5
sudo journalctl -t sudo --since today --no-pager | tail -5
```

Cada uso de `sudo` queda con: quién, desde dónde, como quién, qué comando. Los rechazos dicen `command not allowed` o `user NOT in sudoers`.
