# Comandos — Hardening (guía del instructor)

Cinco minutos, y directo a los labs. Dos cosas que tienen que quedar: (1) SSH se endurece en un archivo de `sshd_config.d/` y **la primera aparición gana**; (2) nunca se cierra la puerta sin haber probado la llave nueva en una sesión aparte. Contraseñas y superficie se explican mientras se hacen.

**Qué es hardening:** endurecer: reducir lo que un atacante puede intentar contra el servidor, y aumentar lo que vamos a ver si lo intenta. No es un producto; es una lista de ajustes.
**Qué es superficie (de ataque):** todo lo que un atacante puede tocar: puertos abiertos, servicios corriendo, cuentas con contraseña, paquetes sin actualizar. Menos superficie, menos riesgo.
**Qué es fuerza bruta:** probar contraseñas una tras otra hasta acertar. Contra SSH expuesto a internet es constante. Con llaves en vez de contraseñas, no hay nada que adivinar.
**Qué es `sshd`:** el servidor SSH (Día 5), el servicio `sshd.service`. Su configuración es `/etc/ssh/sshd_config`.
**Qué es `sshd_config.d/`:** la carpeta de fragmentos: `sshd_config` tiene al principio la línea `Include /etc/ssh/sshd_config.d/*.conf`, que lee todos los archivos `.conf` de esa carpeta, en orden alfabético. RHEL trae `50-redhat.conf`. Lo nuestro va en un archivo nuevo ahí.
**Qué es "la primera aparición gana":** en sshd, si una directiva aparece dos veces, vale la **primera** que se leyó. Como el `Include` está al principio, lo de la carpeta gana sobre lo del archivo principal; y entre archivos de la carpeta, gana el que ordena primero. `50-hardening.conf` ordena antes que `50-redhat.conf` porque `h` viene antes que `r`.
**Qué es una directiva:** una línea de configuración `Nombre valor`, por ejemplo `PermitRootLogin no`.
**Qué es `PermitRootLogin no`:** root no puede entrar por SSH ni con llave ni con contraseña. Se entra como `student` y se usa `sudo` (Día 3).
**Qué es `PasswordAuthentication no`:** sshd no acepta contraseñas: solo llaves.
**Qué es `KbdInteractiveAuthentication no`:** "keyboard-interactive": el otro camino por el que sshd puede pedir una contraseña (a través de PAM). Si solo apagás `PasswordAuthentication`, sshd sigue pidiendo contraseña por este camino. Se apagan los dos.
**Qué es `MaxAuthTries 3`:** tres intentos de autenticación por conexión; después corta.
**Qué es `AllowUsers student`:** solo los usuarios de la lista pueden entrar por SSH; los demás (ana, pedro, root…) reciben `Permission denied` aunque tengan llave. El Día 10 cuenta con esto.
**Qué es `Banner /etc/issue.net`:** el texto que sshd muestra **antes** de autenticar. `/etc/issue.net` es un archivo de texto plano para eso (`/etc/issue` es el equivalente para la consola).
**Qué es `sshd -t`:** "test": revisa la sintaxis de la configuración. Si está bien, no imprime nada. Si está mal, dice archivo y línea.
**Qué es `sshd -T`:** muestra la configuración **efectiva**: lo que sshd va a aplicar después de leer todos los archivos y resolver quién gana. Todo en minúsculas. Es la prueba de que nuestro archivo ganó.
**Qué es `systemctl reload sshd`:** recargar la configuración sin cortar las sesiones abiertas (Día 4: `reload` vs `restart`). Tu sesión sigue viva aunque la nueva configuración te prohibiera entrar.
**Qué es `ssh -o PubkeyAuthentication=no`:** una opción del **cliente** ssh: "no uses mi llave". Sirve para simular a alguien que no tiene llave y ver que no puede entrar.
**Qué es `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`:** la lista entre paréntesis son los métodos que el servidor acepta. Que no aparezca `password` es la prueba de que las contraseñas están apagadas. `gssapi` es autenticación con Kerberos; viene habilitada, no molesta, ignorar.
**Qué es `ssh ... hostname`:** ejecutar un solo comando en el servidor y salir (Día 5). Sirve para probar la conexión sin abrir una sesión.
**Qué es PAM:** "Pluggable Authentication Modules": el sistema de módulos que usan `login`, `su`, `sshd` y `passwd` para autenticar. Se configura en `/etc/pam.d/`. En RHEL 9 **no se edita a mano**: se usa `authselect`.
**Qué es `authselect`:** la herramienta de RHEL 9 para configurar PAM con perfiles. `authselect current` muestra el perfil activo; `authselect enable-feature with-faillock` prende una función del perfil y reescribe `/etc/pam.d/` por vos.
**Qué es el perfil `sssd` (o `local`):** el nombre del perfil de authselect que trae RHEL 9. Cambia con la versión (en 9.x recientes puede decir `local`). No importa cuál: importa que después aparezca `with-faillock`.
**Qué es pwquality:** la biblioteca que revisa la calidad de una contraseña nueva. Su archivo es `/etc/security/pwquality.conf`. Se lee en cada `passwd`. Para root solo avisa (`BAD PASSWORD` y acepta igual, Día 3); para usuarios normales bloquea.
**Qué es `minlen = 12`:** largo mínimo 12.
**Qué es `minclass = 3`:** al menos tres clases de caracteres distintas de cuatro: mayúsculas, minúsculas, dígitos, símbolos.
**Qué es `ocredit = -1`:** "other credit": al menos un carácter que no sea letra ni número (un símbolo). El signo negativo significa "requisito" (positivo sería "bonificación"). Se pone porque `minclass = 3` se cumple sin símbolos.
**Qué es faillock:** el módulo de PAM que cuenta intentos fallidos de autenticación por usuario y bloquea la cuenta un rato. Su archivo es `/etc/security/faillock.conf`.
**Qué es `deny = 5`:** a los cinco intentos fallidos, bloquea.
**Qué es `unlock_time = 900`:** 900 segundos (15 minutos) y se desbloquea solo.
**Qué es `faillock --user X`:** ver los intentos registrados de un usuario. `--reset` los borra y desbloquea. Con `sudo`.
**Qué es `pam_faillock.so preauth / authfail`:** las líneas que authselect agregó a `/etc/pam.d/system-auth`: antes de pedir la contraseña revisa si está bloqueado (`preauth`), y después de fallar anota (`authfail`). Solo se señalan.
**Qué es `ss -tulpn`:** Día 5: qué puertos están escuchando y qué programa. `0.0.0.0` o `*` = escucha para toda la red; `127.0.0.1` = solo desde la propia máquina.
**Qué es `dnf-automatic`:** un paquete que corre `dnf` solo, todos los días, mediante un timer de systemd (Día 7). Su archivo es `/etc/dnf/automatic.conf`: `apply_updates = yes` instala (con `no` solo descarga); `upgrade_type = security` aplica solo parches de seguridad (`default` sería todo). El timer se llama `dnf-automatic.timer`.
**Qué es `/var/log/secure`:** Día 3: el log de autenticación y de `sudo`. Cada `sudo` deja quién, desde dónde y qué comando.

---

## Los cuatro frentes

**Qué decir:** "cuatro frentes, en orden de retorno por minuto invertido. El primero es SSH porque el 90 % de los ataques contra un Linux expuesto son fuerza bruta contra SSH. Con llaves, no hay nada que adivinar."
**Qué señalar:** la última línea: lo que **no** hacemos. "Apagar SELinux o firewalld 'para que funcione' es exactamente lo que el reto prohíbe."

---

## SSH: dónde va la configuración

Leer el diagrama. **La frase que hay que decir:** *"En sshd, la primera vez que aparece una directiva es la que vale. Los archivos de la carpeta se leen primero, en orden alfabético. Por eso lo nuestro va ahí y con un nombre que ordene antes del de Red Hat."*
**Qué señalar en las directivas:** `PasswordAuthentication no` y `KbdInteractiveAuthentication no` van **juntas**; una sola deja la puerta entreabierta. Y `AllowUsers student`: "después de esto, ana y pedro no entran por SSH aunque tengan llave. Es a propósito."
Los tres comandos: `sshd -t` valida, `sshd -T` muestra lo efectivo, `reload` aplica sin cortar. "El orden del Lab 4.1 es la secuencia segura: no se cierra la puerta sin probar la llave nueva en otra ventana."

---

## Contraseñas

Una frase por herramienta: "`pwquality` decide qué contraseña se acepta; `faillock` decide cuántas veces podés equivocarte. Las dos se configuran en `/etc/security/`." El detalle se ve en el Lab 4.2.

---

## Superficie

Una frase: "`ss -tulpn` es la vista del atacante desde adentro: cada `LISTEN` en `0.0.0.0` es una puerta. `cockpit` viene abierto en el firewall y nadie lo pidió: se cierra. Y los parches de seguridad, solos, todos los días."
