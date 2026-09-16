# 4 — Permisos, `umask`, bits especiales y ACL (guía del instructor)

Veinte minutos, el bloque más largo de concepto del día. Es mucha tabla: no las leas enteras. Cada sección tiene una frase que decir y una cosa que señalar; el resto se practica en los labs 4.1 y 4.2.

**Qué es `rwx`:** leer, escribir, ejecutar. Cada archivo tiene tres ternas: para el dueño (`u`), para el grupo (`g`), para los demás (`o`).
**Qué es octal:** escribir cada terna como un número: `r`=4, `w`=2, `x`=1, sumados. `rw-` = 6, `r-x` = 5, `---` = 0. `rw-r--r--` = `644`.
**Qué es `umask`:** un número que se **resta** a los permisos con los que nacería un archivo nuevo. Sin `umask`, todo nacería `666`/`777`; el `umask` quita bits.
**Qué es setuid:** un bit en un programa que hace que corra con los permisos del **dueño del programa**, no del que lo ejecuta. `passwd` es de root y tiene setuid: así un usuario cualquiera puede escribir su contraseña en `/etc/shadow`, que solo root toca.
**Qué es setgid en una carpeta:** todo lo que se crea adentro nace con el **grupo de la carpeta**, no con el grupo primario del que lo creó. Es lo que hace funcionar una carpeta compartida de área.
**Qué es sticky bit:** en una carpeta donde todos escriben, solo el dueño de cada archivo (o root) puede borrarlo. `/tmp` lo tiene: por eso ayer no pudiste borrar las carpetas `systemd-private-*`.
**Qué es una ACL:** una lista de permisos **adicional** al `rwx`, para dar acceso a más de un grupo o a usuarios sueltos sin cambiar el dueño ni el grupo. Se ve como `+` en `ls -l`.
**Qué es la `mask` de una ACL:** el techo. Si el archivo tiene `chmod 600`, la `mask` es `---` y ninguna ACL puede dar más que eso, aunque esté escrita. Se ve en `getfacl` como `#effective:---`.

---

## Leer `ls -l`

`ls -l /etc/hostname /usr/bin/passwd` y `ls -ld /tmp ~`.
**Qué decir:** "ayer leyeron esta línea y les dije 'los permisos son Día 3'. Hoy es Día 3."
**Qué señalar:** `hostname` es `-rw-r--r--`: root escribe, todos leen. `passwd` es `-rwsr-xr-x`: esa `s` es setuid (después). `/tmp` es `drwxrwxrwt`: esa `t` es sticky. Tu home es `drwx------`: solo vos.

**La frase que hay que decir:** *"el sistema mira **una sola** terna. Si sos el dueño, mira `u` y no sigue. Si no, y estás en el grupo, mira `g`. Si no, `o`. No suma: elige."* Consecuencia rara: un archivo `----rwx---` es ilegible para su propio dueño aunque esté en el grupo. Root ignora todo esto.

---

## Octal

**Qué decir:** "es solo otra forma de escribir las mismas nueve letras, con tres números". Hacer uno en voz alta: `rw-r--r--` → 6, 4, 4. Señalar en la tabla `2770` y `1777`: "esos cuatro dígitos son los del Lab 4.2; el primero es el bit especial".

---

## `chmod`

Dos formas, las dos valen. Octal para "dejarlo así"; simbólico para "agregar o quitar una cosa". `chmod u+x` es lo que van a hacer para ejecutar un script. La `X` mayúscula: pone `x` solo a carpetas y a lo que ya era ejecutable — es cómo dar lectura recursiva sin volver ejecutables los documentos. Se usa en el Lab 4.2 con las ACL.

---

## Permisos en una carpeta

**Esto es lo que más confunde. Decirlo despacio:** en una carpeta, `r` es ver los nombres, `x` es poder entrar, `w` es poder crear y **borrar** adentro. *"Borrar un archivo no depende de los permisos del archivo. Depende de los permisos de la carpeta que lo contiene."* Y: *"una carpeta `700` protege todo lo que hay adentro, sin importar qué permisos tenga cada archivo. Por eso en RHEL nadie ve el home de otro."* Se ve con las manos en el Lab 4.1, Parte 4.

---

## `chown` y `chgrp`

Solo root cambia el dueño (si un usuario pudiera regalar archivos, burlaría cuotas y setuid). El dueño puede cambiar el grupo, pero solo a uno al que él pertenece. Dos mensajes distintos que van a ver: `Permission denied` = los `rwx` no lo permiten; `Operation not permitted` = "esto solo lo hace root o el dueño".

---

## `umask`

`umask` y `umask -S`.
**Qué señalar:** `student` tiene `0002`, no `0022`. **No asumir, comprobar.** RHEL da `0002` a los usuarios que tienen grupo privado (el grupo con su nombre): como nadie más está en ese grupo, el `w` de grupo no le abre nada a nadie — hasta que en una carpeta setgid el grupo es el del área, y ahí ese `w` es justo lo que hace que el equipo pueda editar los archivos de los demás. Root tiene `0022`: lo que root crea lo leen todos; si es sensible, `chmod 600` a mano.

---

## Bits especiales

Ya dijiste qué es cada uno arriba. Acá señalar la tabla y el `find -perm -4000`: "estos son los programas del sistema que corren como root aunque los ejecute cualquiera. Son unos 20-25. Un setuid nuevo que ayer no estaba es lo primero que mira un forense."

`S`/`T` mayúsculas: el bit está pero falta el `x`. Es un error de configuración. Lo van a fabricar en el Lab 4.2, Parte 5, para reconocerlo.

---

## ACL

**Qué decir:** "`rwx` da tres opciones: dueño, grupo, resto. Cuando hace falta que **otro** grupo lea — auditoría, que no debe escribir — o que **un** usuario suelto tenga algo distinto, `rwx` no alcanza. Eso es ACL."

**Qué señalar:** `setfacl -m` agrega; `-d -m` agrega **por defecto** (lo que se cree después lo hereda); `-R` aplica a lo que ya existe. Los dos hacen falta: sin `-d`, el auditor lee lo viejo pero no lo nuevo — es el error del Lab 4.2, Parte 7, y del reto. El `+` en `ls -l` avisa que hay ACL; `getfacl` la muestra. Y el límite: una ACL de lectura en un archivo no sirve si el usuario no puede **entrar** a la carpeta.

Cerrar: "en los labs primero permisos normales en su home, después la estructura de la institución en `/srv`: dos carpetas de área con herencia de grupo, un buzón público con sticky, y auditoría con ACL."
