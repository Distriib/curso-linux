# 4 — SSH a fondo (guía del instructor)

Ocho minutos. Lo vienen usando desde el Día 1; hoy se le pone nombre a cada parte y se pasa de contraseña a clave.

**Qué es un par de claves:** dos archivos que se generan juntos y van de la mano. Lo que se cifra con uno solo lo abre el otro. La **privada** se queda en tu computadora y no se copia nunca; la **pública** se reparte.
**Cómo funciona el ingreso por clave:** el servidor tiene tu clave pública. Cuando te conectás, te manda un desafío que solo puede resolver quien tenga la privada. Si lo resolvés, te deja entrar. **La contraseña nunca viaja.**
**Qué es `authorized_keys`:** el archivo en el servidor con la lista de claves públicas autorizadas para ese usuario. Una por línea.
**Qué es `ssh-copy-id`:** un comando que copia tu clave pública al `authorized_keys` del servidor, con los permisos correctos. En Windows no existe y hay que hacerlo a mano.
**Qué es `ed25519`:** el algoritmo de clave recomendado hoy. Antes se usaba `rsa` de 4096 bits, que sigue siendo válido en sistemas viejos.
**Qué es la passphrase:** una contraseña que protege la clave privada por si alguien te roba el archivo. En el curso se deja vacía; en producción se pone y se usa `ssh-agent` para no escribirla cada vez.
**Qué es `known_hosts`:** el archivo en **tu** computadora donde se guarda la huella de cada servidor al que entraste. Sirve para detectar si alguien suplantó el servidor.
**Qué es la huella (fingerprint):** un resumen corto de la clave pública del servidor. Se compara para verificar que el servidor es el que decís.
**Qué es `scp`:** copiar archivos por SSH. **Qué es `rsync`:** sincronizar carpetas por SSH, enviando solo lo que cambió.
**Qué es un túnel:** SSH lleva, además de tu terminal, cualquier otro tráfico. Un puerto de tu máquina queda conectado a un puerto visto desde el servidor.

---

## Qué decir y qué señalar

**Contraseña vs clave** — *"hasta hoy escribieron la contraseña cada vez. Con clave, no la escriben nunca más, y además es más seguro: la contraseña se puede adivinar a fuerza bruta; una clave, no"*.

**La privada no se copia.** Decirlo con énfasis: *"si alguna vez alguien les pide el archivo sin `.pub`, les está pidiendo la llave de su casa"*.

**Los permisos.** El detalle que arruina el lab: si `~/.ssh` o `authorized_keys` tienen permisos de más, `sshd` **ignora la clave sin decir nada** y sigue pidiendo contraseña. El motivo solo aparece en el log del servidor. Frase: *"si copiaron la clave a mano y sigue pidiendo contraseña, miren los permisos antes que ninguna otra cosa"*.

**`-p` vs `-P`** — en `ssh` el puerto es `-p` minúscula; en `scp` y `sftp` es `-P` mayúscula. Es un error clásico y lo van a cometer.

**La barra final de `rsync`** — con barra copia el contenido; sin barra copia la carpeta adentro del destino. Vale la pena mostrarlo una vez.

**El túnel** — el ejemplo del lab es llegar a un servidor web en el puerto 8000 de la VM que el firewall **no** deja entrar. El túnel entra por el 22, que sí está permitido, y desde adentro de la VM habla con su propio `localhost`. Frase: *"es el truco para llegar a consolas internas y bases de datos sin abrir un puerto más en el firewall"*.

## Lo que se mira pero no se toca

La configuración del servidor SSH vive en `/etc/ssh/sshd_config` y en `/etc/ssh/sshd_config.d/`. El endurecimiento (prohibir root, prohibir contraseñas) es el **Día 8**. Hoy solo se menciona que existe.
