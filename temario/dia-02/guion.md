# Día 2 — Guion de clase

Qué hacés, minuto a minuto. Cada bloque dice: **qué decir**, **qué escribir**,
y **cómo sabés que salió bien**.

La estructura de todo el día es siempre la misma:
**vos explicás poco → vos escribís un comando → ellos lo escriben → miran su resultado.**

---

## 0:00–0:10 · Repaso

**Qué hacés:** que todos se conecten y comprobar que el Día 1 quedó bien.

Pediles que corran esto y peguen la salida en el chat:

```bash
hostnamectl
df -h /
```

**Tres preguntas del Día 1:**
1. ¿Con qué comando sé quién soy?
2. ¿Por qué tuvimos que registrar la máquina?
3. ¿Qué pasa si escribo la contraseña y no veo nada?

**Sale bien si:** los cuatro pegaron la salida y dice `rhel01`.

---

## 0:10–0:30 · Concepto: la shell y el árbol

**Qué decir, en dos partes de 10 minutos.**

**Parte 1, la shell.** Que la shell es un programa que interpreta lo que
escriben. Un comando tiene tres partes: `ls -l /etc` es comando, opción y
argumento. Que hay atajos y memoria: Tab completa, flecha arriba repite.

**Parte 2, el árbol.** Acá abrís [arbol-rhel.md](arbol-rhel.md) y lo contás con
la terminal al lado. Lo importante: en Windows hay letras de disco, en Linux
todo cuelga de `/`. Y las tres carpetas que van a pisar todo el curso: `/etc`
es la configuración, `/var/log` es qué pasó, `/home` es la gente.

**Mostralo mientras hablás:**

```bash
ls -l /
```

**No hagas diapositivas.** Señalá las carpetas en la pantalla.

---

## 0:30–0:50 · Lab 1.1 — La shell por dentro

**Qué hacés:** escribís cada comando, explicás qué salió, ellos repiten.

```bash
echo $PATH
echo $HOME
echo $USER
env | wc -l
```

**Qué decir:** `PATH` es la lista de carpetas donde la shell busca los
programas. Por eso escribís `ls` y funciona desde cualquier lado.

Después, variables propias:

```bash
MIVAR=hola
echo $MIVAR
bash
echo "Hija: $MIVAR"     # sale vacío
exit
export MIVAR
```

**Qué decir:** una variable normal no la heredan los programas hijos. Con
`export` sí. Eso explica muchos errores de scripts en la Jornada 7.

**Checkpoint:** pegar la salida de `echo $PATH`.

---

## 0:50–1:05 · Concepto: archivos, `ls -l` y enlaces

**Qué decir.** Abrí `ls -l` en pantalla y explicá **columna por columna**:
tipo, permisos, dueño, grupo, tamaño, fecha, nombre.

Los permisos solo se **leen** hoy. Cambiarlos es el Día 3. Decilo en voz alta
para que nadie se pierda esperando.

Después: la diferencia entre `cp` y `cp -a`, el peligro de `rm -r`, y qué es un
enlace (un atajo a otro archivo).

---

## 1:05–1:30 · Lab 2.1 — Construir `~/empresa`

**Qué hacés:** construyen la carpeta sobre la que van a trabajar el resto del día.

```bash
mkdir -p ~/empresa/{documentos,clientes,backups,logs}
tree ~/empresa
cd ~/empresa/documentos
for i in {1..5}; do echo "Informe numero $i de PanamaTech" > informe$i.txt; done
touch plan-2026.doc presupuesto.csv
ls -l
```

Después las variantes de `ls`, que es donde se entiende para qué sirve cada una:

```bash
ls -lh      # tamaños legibles
ls -lt      # por fecha
ls -lS      # por tamaño
ls -R       # todo el árbol
```

Y copiar, mover, enlazar.

**Checkpoint:** pegar la salida de `ls -R ~/empresa`.

---

## 1:30–1:45 · Concepto: grep, tuberías y redirección

**Esta es la parte más importante del día.** Decilo así.

**Qué decir:** un administrador no lee logs, los filtra. Un servidor genera
50.000 líneas por día y la que importa es una.

Tres ideas, nada más:

- `>` guarda la salida en un archivo
- `|` pasa la salida de un comando al siguiente
- `grep` deja pasar solo las líneas que coinciden

Mostralo con un ejemplo de una línea:

```bash
ls /etc | grep ssh
```

---

## 1:45–2:00 · Descanso

---

## 2:00–2:30 · Lab 3.1 — Analizar un log de 300 líneas

**El log no existe: lo generan.** Les pasás el bloque `genera-log.sh` del
material, lo pegan tal cual y lo ejecutan:

```bash
bash ~/empresa/genera-log.sh > ~/empresa/logs/servidor.log
wc -l ~/empresa/logs/servidor.log
md5sum ~/empresa/logs/servidor.log
```

**Qué decir:** el `md5sum` tiene que coincidir con el tuyo. Así todos tienen el
mismo archivo y los números van a dar igual.

Después, la secuencia que enseña el día entero:

```bash
grep ERROR ~/empresa/logs/servidor.log
grep -c ERROR ~/empresa/logs/servidor.log
grep ERROR ~/empresa/logs/servidor.log > ~/empresa/logs/errores.txt
cut -d' ' -f4 ~/empresa/logs/servidor.log | sort | uniq -c | sort -rn
```

**Qué decir en el último:** "extraé la columna 4, ordenala, contá repetidos, y
ordená por cantidad". Eso responde *quién generó más entradas*. Es la consulta
que van a hacer toda su vida.

**Checkpoint:** ¿cuántas líneas ERROR les salieron?

⚠️ Antes de la clase: generá el log y anotá los números reales.

---

## 2:30–2:40 · Concepto: buscar

**Qué decir:** `grep` busca **dentro** de los archivos. `find` busca **los
archivos**. Es la diferencia clave.

`find` busca por nombre, tamaño, fecha, dueño o permisos. Y puede ejecutar algo
sobre lo que encuentra.

---

## 2:40–2:55 · Lab 4.1 — `find`

```bash
sudo find /etc -name "*.conf" | wc -l
sudo find /var/log -type f -size +1M
sudo find /etc -type f -mmin -120 | head
find /home -user student -type f | wc -l
find /usr/bin -perm -4000 -type f | head -5
```

**Qué decir en el último:** esos son programas que corren con permisos de root
aunque los lance un usuario normal. Es la lista que revisa cualquiera que
audite un servidor.

**Checkpoint:** ¿cuántos `.conf` hay en `/etc`?

---

## 2:55–3:15 · Concepto + Lab 5.1 — Respaldos

**Qué decir:** `tar` junta muchos archivos en uno. La compresión es aparte.

```bash
cd ~/empresa/backups
seq 1 300000 > numeros.txt
ls -lh numeros.txt
gzip -k numeros.txt
bzip2 -k numeros.txt
ls -lh numeros.txt*
```

**Qué decir:** comparen los tamaños. Más compresión, más tiempo de CPU.

Y el respaldo de verdad:

```bash
tar -czf /tmp/empresa-$(date +%F).tar.gz -C ~ empresa
tar -tzf /tmp/empresa-$(date +%F).tar.gz | head
```

**Qué decir:** `c` crea, `z` comprime, `f` es el archivo. Y `t` lista sin
extraer, que es cómo se verifica un respaldo antes de confiar en él.

---

## 3:15–3:30 · Concepto + Lab 6.1 — vim

**Qué decir primero, para bajar la ansiedad:** vim tiene dos modos y toda la
confusión viene de ahí. Al abrirlo estás en modo **comando**: las teclas no
escriben, ejecutan. Se aprieta `i` para escribir y `Esc` para volver.

**Lo mínimo, y nada más:**

| Tecla | Qué hace |
|---|---|
| `i` | Empezar a escribir |
| `Esc` | Dejar de escribir |
| `:w` | Guardar |
| `:q` | Salir |
| `:wq` | Guardar y salir |
| `:q!` | Salir sin guardar |

Crean el archivo y lo editan:

```bash
vim ~/empresa/documentos/app.conf
```

El ejercicio: cambiar `srv-old` por `srv-new` en las tres líneas donde aparece.

```bash
grep -n "srv-" ~/empresa/documentos/app.conf
```

---

## 3:30–3:50 · Reto individual

Les pasás un log de 200 líneas. Tienen que contestar solos:

1. ¿Cuántas líneas tienen `FAIL`?
2. ¿Cuál es la última línea con `WARN`?
3. Crear un archivo solo con las líneas de `ana`
4. Empaquetarlo con la fecha en el nombre

**Vos no ayudás.** Solo mirás quién se traba y en qué.

---

## 3:50–4:00 · Cierre

Tres preguntas al aire:

1. ¿Cómo cuento cuántas veces aparece una palabra en un archivo?
2. ¿Cuál es la diferencia entre `grep` y `find`?
3. ¿Cómo verifico que un respaldo quedó bien?

Y el snapshot `dia02-fin`.

---

## Lo que tenés que hacer antes de la clase

1. Correr el Lab 3.1 completo y **anotar los números reales** del log
2. Correr el Lab 4.1 y anotar cuántos `.conf` hay en `/etc`
3. Tener [arbol-rhel.md](arbol-rhel.md) abierto en otra ventana
4. Tener este guion abierto en el celular o en una segunda pantalla
