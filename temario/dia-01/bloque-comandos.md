# Bloque de comandos — 45 min

## Qué vamos a hacer

Una radiografía del servidor. Cuatro rondas, cada una responde una pregunta
sobre **su propia** máquina.

No es una lista de comandos para memorizar. El instructor escribe, ellos
escriben lo mismo, y después **cada uno lee su propia salida y contesta**.
Eso es lo que llena los 45 minutos: no el tecleo, sino leer el resultado.

Cada ronda cierra pegando una salida en el chat.

---

## Ronda 1 — ¿Quién soy? (10 min)

```bash
whoami
id
pwd
hostname
```

**Preguntas para el grupo:**

- ¿Qué número les salió en `uid=`? ¿Es el mismo para todos?
- En `id`, ¿a qué grupos pertenecen? ¿Ven `wheel` ahí?
- `pwd` dice `/home/student`. ¿Por qué el prompt muestra `~` y no esa ruta?

**Qué explicar:** `wheel` es el grupo que da permiso de `sudo`. Está ahí porque
marcaron "Make this user administrator" en la instalación. El `~` es un atajo
que significa "mi carpeta personal".

**Checkpoint:** pegar la salida de `id`.

---

## Ronda 2 — ¿Qué máquina es esta? (12 min)

```bash
hostnamectl
uname -r
uname -m
lscpu | head -15
uptime
date
```

**Preguntas:**

- ¿Qué versión de RHEL dice `hostnamectl`?
- ¿Qué arquitectura les salió en `uname -m`?
- ¿Cuántos núcleos ven en `lscpu`?

**Momento clave:** aquí sale la diferencia de arquitectura. El instructor tiene
`aarch64` y ellos `x86_64`. Anunciarlo, explicar por qué (Mac con chip ARM), y
aclarar que todos los comandos son idénticos.

**Qué explicar:** `hostnamectl` es el resumen de identidad de la máquina.
`uname -r` es el kernel, que no es lo mismo que la versión de RHEL.

**Checkpoint:** pegar la salida de `hostnamectl`.

---

## Ronda 3 — ¿Cómo está de recursos? (12 min)

```bash
df -h
free -m
lsblk
ip a
ping -c 3 redhat.com
```

**Preguntas:**

- ¿Cuánto espacio libre les queda en `/`?
- En `free -m`, ¿cuánta memoria total dice? ¿No eran 4096?
- En `lsblk`, ¿cómo se llama su disco?
- ¿Qué IP tienen?

**Momento clave:** `free -m` no da 4096 sino unos 3800. Dejar que lo noten y
preguntar por qué. Respuesta: **kdump** reservó memoria para poder guardar un
volcado si el kernel se cae.

Y en `lsblk` aparece la otra diferencia: `sda` en VirtualBox, `vda` en UTM.

**Checkpoint:** pegar la salida de `df -h`.

---

## Ronda 4 — ¿Cómo me defiendo solo? (11 min)

Esta es la más importante del bloque: aprenden a no depender del instructor.

```bash
man ls
```

Dentro del manual: flechas para moverse, `/permisos` para buscar, `q` para salir.

```bash
ls --help | head -20
man -k hostname
```

Atajos, probándolos en vivo:

- `Tab` — escribir `hostn` y apretar Tab
- `↑` — recuperar comandos anteriores
- `Ctrl+L` — limpiar pantalla
- `history` — ver todo lo escrito

Y sudo:

```bash
cat /etc/shadow        # Permission denied
sudo !!                # el mismo comando, con permisos
```

**Qué explicar:** `sudo !!` repite el comando anterior con permisos. Es el atajo
que más van a usar en su vida. Y `man` es la razón por la que un administrador
puede trabajar sin internet.

**Checkpoint:** que cada uno encuentre con `man` para qué sirve la opción `-h`
de `df`, y lo escriba en el chat con sus palabras.

---

## Si sobra tiempo

```bash
ls -l /
tree -L 1 /
cd /etc && ls | head -20
cd -
```

Recorrido rápido por la raíz, nombrando qué guarda cada carpeta. Es el arranque
natural del Día 2, así que si no da el tiempo no se pierde nada.

---

## Cierre del bloque

Antes de pasar al snapshot, tres preguntas al aire:

1. ¿Cómo averiguo con qué usuario estoy trabajando?
2. ¿Cómo veo cuánto disco me queda?
3. Si no me acuerdo de una opción, ¿qué hago?

Si contestan las tres, el bloque cumplió.
