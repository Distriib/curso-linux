# Comandos — Hardening

Endurecer = reducir lo que un atacante puede intentar, y ver si lo intenta. Cuatro frentes, en orden de retorno por minuto:

| Frente | Qué se hace | Con qué |
|---|---|---|
| 1. Acceso remoto (SSH) | solo llaves, root afuera, lista de usuarios, pocos intentos | `/etc/ssh/sshd_config.d/` |
| 2. Contraseñas | complejidad, bloqueo por intentos fallidos | `pwquality`, `faillock` |
| 3. Superficie | servicios que nadie usa, puertos "que venían así", parches solos | `ss`, `firewall-cmd`, `dnf-automatic` |
| 4. Visibilidad | lo que ya se registra | `/var/log/secure`, `ausearch` |

Lo que **no** hacemos: apagar SELinux ni firewalld "para que funcione".

## SSH: dónde va la configuración

```
/etc/ssh/sshd_config                       ← línea ~19: Include /etc/ssh/sshd_config.d/*.conf
/etc/ssh/sshd_config.d/50-redhat.conf      ← lo que trae RHEL
/etc/ssh/sshd_config.d/50-hardening.conf   ← lo nuestro (h < r: se lee antes)
```

En sshd, **la primera aparición de una directiva gana**. Los archivos de `sshd_config.d/` se leen al principio, en orden alfabético. Por eso lo nuestro va ahí, con un nombre que ordene antes.

| Directiva | Qué hace |
|---|---|
| `PermitRootLogin no` | root no entra por SSH |
| `PasswordAuthentication no` | sin contraseña: solo llave |
| `KbdInteractiveAuthentication no` | cierra la otra puerta por la que sshd pide contraseña |
| `MaxAuthTries 3` | tres intentos por conexión |
| `AllowUsers student` | solo estos usuarios (los demás, ni con llave) |
| `Banner /etc/issue.net` | texto legal antes de autenticar |

| Comando | Qué hace |
|---|---|
| `sudo sshd -t` | revisa la sintaxis (no imprime nada = bien) |
| `sudo sshd -T` | la configuración **efectiva**, ya resueltos todos los archivos |
| `sudo systemctl reload sshd` | aplicar sin cortar las sesiones abiertas |

## Contraseñas

| Archivo / comando | Qué hace |
|---|---|
| `/etc/security/pwquality.conf` | reglas de calidad: `minlen = 12`, `minclass = 3`, `ocredit = -1` |
| `sudo authselect enable-feature with-faillock` | activa el bloqueo por intentos fallidos |
| `/etc/security/faillock.conf` | umbrales: `deny = 5`, `unlock_time = 900` |
| `sudo faillock --user pedro` | ver los intentos fallidos de alguien |
| `sudo faillock --user pedro --reset` | desbloquearlo |

Para root, `pwquality` solo avisa; para los usuarios normales, bloquea.

## Superficie

| Comando | Qué hace |
|---|---|
| `systemctl list-units --type=service --state=running` | qué corre |
| `sudo ss -tulpn` | qué escucha (la vista del atacante) |
| `sudo firewall-cmd --permanent --remove-service=cockpit` | cerrar lo que nadie pidió |
| `/etc/dnf/automatic.conf` + `dnf-automatic.timer` | parches de seguridad solos, todos los días |
