# 4 — Stratis (guía del instructor)

Cinco minutos. No se corre nada: el lab instala y arma todo. Dos cosas que tienen que quedar: (1) Stratis hace en un comando lo que LVM en varios, con una tabla de equivalencias; (2) `df` en Stratis miente porque el espacio es virtual: la verdad la dice `stratis pool list`.

**Qué es Stratis:** una herramienta de Red Hat para administrar almacenamiento con menos pasos que LVM. Tiene dos partes: `stratisd`, el servicio que hace el trabajo, y `stratis`, el comando con el que se le habla. Por debajo usa las mismas piezas del kernel que LVM (device mapper) y XFS.
**Qué es un demonio / servicio:** un programa que corre de fondo todo el tiempo (Día 4: `systemctl`). `stratisd` tiene que estar corriendo para que `stratis` funcione; si no, dice `Failed to connect to stratisd`.
**Qué es `enable --now`:** Día 4: arrancar el servicio ahora y dejarlo habilitado para el próximo arranque.
**Qué es un pool:** el conjunto de discos que administra Stratis. Equivale a PV + VG en LVM. Se crea con un disco o partición y se amplía agregando otros.
**Qué es un filesystem en Stratis:** un XFS que Stratis crea dentro del pool. No hace falta `mkfs`: ya viene formateado. Aparece en `/dev/stratis/pool1/fs1`.
**Qué es thin provisioning (aprovisionamiento delgado):** presentar un tamaño mayor al que existe físicamente y gastar el espacio real a medida que se escribe. Cada filesystem de Stratis se presenta con 1 TiB aunque el pool tenga 3 GiB. La ventaja: no hay que decidir tamaños. El riesgo: `df` dice que sobra espacio cuando el pool puede estar lleno; si el pool se llena, fallan las escrituras de todos los filesystems del pool.
**Qué es `stratis pool list`:** total, ocupado y libre **reales** del pool. Es lo que hay que vigilar en vez de `df`.
**Qué es `~Ca,~Cr, Op`:** propiedades del pool: `~Ca` sin caché, `~Cr` sin cifrado, `Op` permite dar más espacio virtual que el físico. No hay que explicarlas; están ahí.
**Qué es una firma y `wipefs -a`:** Stratis exige discos sin ninguna firma (ni sistema de archivos ni LVM). Si la partición tuvo algo, `wipefs -a` la limpia. Lo mismo del Bloque 2.
**Qué es `add-data`:** agregar un disco o partición al pool. El pool crece al instante; los filesystems no cambian porque ya "miden" 1 TiB.
**Qué es `blockdev list`:** los discos que forman el pool, con su tamaño y su rol (`Data`).
**Qué es un snapshot de Stratis:** una copia del filesystem en ese instante, que se comporta como un filesystem más: se monta, se escribe. Tiene UUID propio, así que no necesita `nouuid`. Se borra con `filesystem destroy`.
**Qué es `x-systemd.requires=stratisd.service`:** opción de `fstab` que le dice a systemd: "antes de montar esto, esperá a que `stratisd` esté arrancado". Sin eso, al arrancar el dispositivo `/dev/stratis/...` todavía no existe y la VM cae en modo de emergencia.
**Qué es `/dev/mapper/stratis-1-...-thin-fs-...`:** el nombre largo que muestran `df` y `findmnt`: el dispositivo real que Stratis armó con device mapper. `/dev/stratis/pool1/fs1` es el enlace corto que usamos nosotros.

---

## La tabla LVM ↔ Stratis

Leerla fila por fila. **Qué decir:** *"lo mismo que hicimos en LVM con cinco comandos, Stratis lo hace con dos: `pool create` y `filesystem create`. Ampliar es `add-data`, y listo: no hay `lvextend`, no hay `xfs_growfs`."*

---

## Comandos

No leer toda la tabla. Señalar `pool create`, `filesystem create`, `add-data`, y `wipefs` (*"si la partición tuvo algo antes, Stratis se niega; se limpia con esto"*).

---

## `df` dice 1 TiB

**La frase:** *"Stratis le muestra al sistema un disco de 1 TiB que no existe. Va gastando el espacio real a medida que escriben. `df` va a decir que sobra; la verdad la dice `stratis pool list`. Si alguien copia 4 GiB en un pool de 3, las escrituras fallan aunque `df` diga que hay 1000 GiB libres."* Preguntarles eso mismo en el lab.

---

## La línea de `fstab`

**Qué señalar:** la opción `x-systemd.requires=stratisd.service`: *"sin esto, el próximo arranque cae en modo de emergencia, porque el dispositivo aparece solo cuando `stratisd` arranca."* Es la opción de `fstab` más olvidada del día.

Del examen: Stratis ya no está en la lista de objetivos del RHCSA para RHEL 9; se enseña porque está en la ficha del curso y porque es la vía que Red Hat propone para quien no quiere administrar LVM a mano.
