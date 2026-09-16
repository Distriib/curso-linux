# 1 — Quién es quién (guía del instructor)

Quince minutos, tipeando mientras explicás. Los comandos están en la pestaña de estudiantes. No se modifica nada hoy en este bloque: solo se lee.

**Qué es un UID:** el número con el que el sistema identifica a un usuario. **GID:** lo mismo para un grupo. El sistema trabaja con números; los nombres son para las personas.
**Qué es root:** la cuenta con UID 0. El sistema no le aplica ninguna verificación de permisos. Cualquier cuenta con UID 0 sería root aunque se llamara distinto — por eso se audita `/etc/passwd`.
**Qué es una cuenta de sistema:** una cuenta que no es de una persona sino de un programa (`sshd`, `chrony`). UID menor a 1000, sin contraseña, con shell `nologin`. Ya las vieron ayer en `tail /etc/passwd`.
**Qué es `nobody`:** la cuenta de sistema para "no ser nadie": UID 65534, sin contraseña, sin home, `nologin`, sin permisos sobre nada. La usan programas que no necesitan privilegios (NFS, por ejemplo) para que, si los comprometen, el atacante no pueda nada. Va a salir en el Lab 1.1, Parte 4, porque es la única cuenta de sistema con UID alto — a propósito, fuera del rango de personas. No es el dueño de los archivos huérfanos (esos muestran un número). No se borra.
**Qué es el hash:** la contraseña cifrada de forma irreversible. `$6$` es el algoritmo (SHA-512). El sistema no guarda la contraseña; guarda el hash y compara.
**Qué es `getent`:** consulta cuentas o grupos por su nombre. Hace lo mismo que `grep` en `/etc/passwd`, pero también funciona si los usuarios vienen de un directorio corporativo (Active Directory). Por eso se enseña este y no `grep`.
**Grupo primario vs suplementario:** el primario es "con qué grupo nacen mis archivos" (campo 4 de `passwd`). Los suplementarios son "a qué otros grupos pertenezco" (lista en `/etc/group`). Un usuario tiene un solo primario y cualquier cantidad de suplementarios.

---

## Para el sistema no hay nombres, hay números

`id` e `id root`.
**Qué decir:** "el sistema no sabe quién es `student`; sabe que hay un UID 1000. Los nombres son traducción".
**Qué señalar:** `uid=1000(student) gid=1000(student) groups=1000(student),10(wheel)`. UID 1000 = primer usuario normal. `gid=1000(student)` = su grupo primario es su grupo privado. `10(wheel)` = suplementario, el grupo de administradores de RHEL; por eso `student` puede usar `sudo`. El `context=...` es SELinux, Día 8. En `id root`: `uid=0`.

La tabla de rangos: no leerla, señalar las dos filas que importan — `0` es root, `1000+` son las personas. Todo lo del medio son servicios.

---

## `/etc/passwd`

`getent passwd student root`.
**Qué decir:** "siete campos separados por dos puntos. Ayer lo cortaron con `cut -d:`; hoy sabemos qué es cada pedazo".
**Qué señalar:** el campo 2 es una `x` — ahí estaba la contraseña hace 30 años; hoy dice "buscala en shadow". El campo 4 (`1000`) es el GID del grupo primario. El campo 7 es la shell: `/bin/bash` puede entrar; `/sbin/nologin` no. Este archivo lo puede leer cualquiera (`ls -l` lo necesita para traducir UID a nombre) — por eso las contraseñas están en otro lado.

---

## `/etc/shadow`

`sudo grep student /etc/shadow`.
**Qué decir:** "nueve campos. El segundo es la contraseña cifrada; del tercero al octavo son fechas y plazos, en días contados desde el 1 de enero de 1970".
**Qué señalar:** el hash empieza con `$6$` — SHA-512, no es reversible. Los campos `0:99999:7` son: puede cambiarla cuando quiera, nunca expira, avisaría 7 días antes. Los tres últimos vacíos. **Eso es "sin política"** — es el default de RHEL y es lo que van a corregir en el Lab 2.2.

La tabla del campo 2: `!!` es lo que queda después de `useradd` (sin contraseña aún); `!` delante del hash es "bloqueada" — el hash sigue ahí, al desbloquear vuelve. Lo van a ver con sus manos en el Lab 2.2.

---

## `/etc/group`

`getent group wheel student`.
**Qué señalar:** `wheel:x:10:student` — el cuarto campo es la lista de miembros suplementarios. `student:x:1000:` sale **vacío** porque el grupo primario no se anota acá, se anota en `passwd`. Es la pregunta que va a salir: "¿por qué el grupo `student` no tiene a `student`?".

**Qué decir del grupo privado:** "RHEL le crea a cada usuario un grupo con su nombre y nadie más adentro. Parece inútil; en el Bloque 4 van a ver que es lo que permite un `umask` más permisivo sin riesgo".

---

## Los cuatro archivos y sus permisos

`ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow`.
**Qué señalar:** `passwd` y `group` son `-rw-r--r--` (todos leen). `shadow` y `gshadow` son `----------`: **nadie** tiene permiso, ni root. Root los lee porque a UID 0 no se le aplican los permisos — es la demostración de lo que dijiste al principio. `gshadow` es el `shadow` de los grupos (contraseñas de grupo, casi no se usa).

---

## Quién está y quién estuvo

No demostrar todos; se hacen en el lab. Una frase: "`who` y `w` son ahora; `last` es la historia; `lastb` son los intentos fallidos — mañana con los logs los vamos a ver llenos".
