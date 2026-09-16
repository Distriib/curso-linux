# 3 — Privilegios: `su`, `sudo` y `sudoers` (guía del instructor)

Diez minutos. Tres cosas que tienen que quedar: (1) `sudo` registra quién hizo qué y `su -` no; (2) `sudoers` se edita solo con `visudo` y las reglas propias van en `sudoers.d/`; (3) la anatomía de una regla. Lo demás lo practican.

**Qué es `su`:** *switch user*. Cambia de usuario dentro de la sesión. Pide la contraseña del usuario **destino**. `su -` (con guion) es "como root"; `su - ana` es "como ana".
**Qué es `sudo`:** ejecuta **un** comando como root (o como otro usuario), pidiendo **tu propia** contraseña, y anota en el log quién, cuándo y qué. Solo funciona si hay una regla que te lo permita.
**Qué es `/etc/sudoers`:** el archivo con esas reglas. Quién puede ejecutar qué, como quién, en qué servidor.
**Qué es `visudo`:** el editor obligatorio para `sudoers`. Abre el archivo (con `vi`), y al guardar **revisa la sintaxis**: si hay un error, no deja guardar roto. Es la única protección contra dejar a todos sin `sudo`.
**Qué es `/etc/sudoers.d/`:** una carpeta donde cada delegación va en su propio archivo. `sudoers` los carga por la línea `#includedir /etc/sudoers.d` — ese `#` **no es comentario**. Ventaja: se revoca borrando el archivo, y las actualizaciones no lo tocan.
**Qué es `wheel`:** el grupo de administradores de RHEL. `sudoers` trae `%wheel ALL=(ALL) ALL`: quien esté en `wheel` puede todo con su contraseña. `student` está ahí porque marcaron "administrador" al instalar. Dar sudo total a alguien = `usermod -aG wheel usuario`.

---

## Tres caminos hacia root

**Qué decir:** "los tres llegan al mismo lugar. La diferencia es qué queda escrito. Con `su -` el log dice 'alguien se hizo root' y no sabés quién ni qué hizo. Con `sudo` el log dice `pedro ejecutó systemctl restart chronyd a las 10:31`. En una auditoría, eso es la diferencia entre tener respuesta y no tenerla."

En RHEL 9, root no puede entrar por SSH con contraseña (lo vieron en el instalador). Bien.

---

## `su` vs `su -`

Los dos `su ... -c 'pwd; echo $HOME'`.
**Qué señalar:** sin guion, `pwd` da `/home/student` — ana "aparece" pero seguís parado en tu carpeta con tu entorno. Con guion, `/home/ana`: es una sesión de verdad de ana. **Regla: siempre el guion.** Hoy en los labs se usa mucho `sudo su - usuario`: root no necesita contraseña para hacerse otro usuario, así no escriben `Pgn.2026` cincuenta veces.

---

## `sudo` — uso diario

Señalar dos: `sudo -l` ("¿qué puedo hacer?") y `sudo -l -U pedro` ("¿qué puede hacer pedro?" — auditar sin saber su clave). `sudo -i` lo conocen. `sudo -k` para el detalle de que la contraseña se guarda 5 minutos.

---

## `/etc/sudoers`

`sudo grep wheel /etc/sudoers`, `sudo grep includedir /etc/sudoers` y el `ls -l`.
**Qué señalar:** la línea `%wheel ALL=(ALL) ALL` (por eso `student` puede todo). La línea `#includedir /etc/sudoers.d` — **decir que ese `#` no es comentario**, es una directiva. `sudoers` es `-r--r-----`: ni root lo escribe directo, lo hace `visudo` por él. `sudoers.d` está vacío: en el lab le ponen el primer archivo.

**Decir completo:** *"si guardan un `sudoers` con un error de sintaxis, `sudo` deja de funcionar para **todos**, incluido ustedes. La salida es `su -` con la clave de root y borrar el archivo. Por eso: `visudo` siempre, nunca `vim /etc/sudoers`."*

---

## Anatomía de una regla

Leer el diagrama de izquierda a derecha, señalando cada pieza: quién (`%soporte` = el grupo; sin `%` sería un usuario), dónde (`ALL` = cualquier servidor; importa si el archivo se copia a varios), como quién (`(root)`), qué (ruta **completa** `/usr/bin/systemctl`, y los argumentos).

**Los dos detalles que hay que decir:**
- Si escribís argumentos, `sudo` exige que coincidan **exactamente**. `systemctl restart chronyd` permitido ≠ `systemctl restart chronyd --now`. Si no escribís argumentos, vale cualquiera — que es más peligroso.
- **Nunca delegar un comando que abra una shell o un editor.** `sudo vim x` → dentro de vim `:!bash` → shell de root. `less`, `find -exec`, `python`, igual. Es lo primero que busca un atacante con `sudo -l`.

---

## Auditoría

Los dos `grep`/`journalctl`. Ahora van a estar casi vacíos (solo tus `sudo` de hoy). Decir: "cada línea dice quién, desde dónde, como quién y qué. En el lab van a generar líneas de rechazo y las van a leer."
