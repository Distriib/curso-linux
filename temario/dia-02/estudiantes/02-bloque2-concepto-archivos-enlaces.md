# 2 — Archivos, directorios y enlaces

## Leer `ls -l`

```bash
ls -l /etc/hostname
```

```
-rw-r--r--. 1 root root 7 Sep 14 18:02 /etc/hostname
│└──┬────┘│ │  │    │   │      │         └ nombre
│   │     │ │  │    │   │      └ fecha de última modificación
│   │     │ │  │    │   └ tamaño en bytes (-h lo muestra en K/M/G)
│   │     │ │  │    └ grupo dueño
│   │     │ │  └ usuario dueño
│   │     │ └ cantidad de enlaces duros
│   │     └ punto = tiene contexto SELinux
│   └ permisos (Día 3)
└ tipo de archivo
```

### Tipos de archivo (primer carácter)

```bash
ls -l /etc/hostname /bin /dev/null /dev/sda    # en UTM el disco es /dev/vda
```

| Letra | Tipo |
|---|---|
| `-` | archivo normal |
| `d` | directorio |
| `l` | enlace simbólico |
| `c` | dispositivo de caracteres (`/dev/null`) |
| `b` | dispositivo de bloques (`/dev/sda`) |

### Opciones útiles de `ls`

| Opción | Qué hace |
|---|---|
| `-a` | incluye ocultos (`.algo`) |
| `-h` | tamaños legibles |
| `-t` | ordena por fecha (reciente primero) |
| `-S` | ordena por tamaño |
| `-R` | recursivo |
| `-d` | el directorio en sí, no su contenido |
| `-i` | muestra el inodo |

---

## Crear, copiar, mover, borrar

```bash
cd /tmp
mkdir -p demo/a/b
touch demo/x.txt
echo "hola" > demo/x.txt
cp demo/x.txt demo/y.txt
mv demo/y.txt demo/z.txt
ls -l demo
```

| Comando | Qué hace | Opciones clave |
|---|---|---|
| `mkdir -p a/b/c` | crea toda la cadena | |
| `touch archivo` | crea vacío o actualiza fecha | |
| `cp origen destino` | copia | `-r` directorios, `-i` pregunta, `-v` cuenta, `-a` conserva todo |
| `mv origen destino` | mueve **o renombra** (es lo mismo) | `-i` pregunta, `-v` cuenta |
| `rm archivo` | borra. **No hay papelera** | `-i` pregunta, `-r` directorios, `-f` fuerza |
| `rmdir carpeta` | borra la carpeta solo si está vacía | |

```bash
rm -i demo/z.txt
rmdir demo/a
rm -r demo/a
ls demo
```

> Antes de un `rm -r`, hacer `ls` de la misma ruta. Un espacio de más y borrás otra cosa.

---

## Enlaces

```bash
ln demo/x.txt demo/x-duro.txt
ln -s /tmp/demo/x.txt demo/x-link.txt
ls -li demo
```

| | Duro `ln` | Simbólico `ln -s` |
|---|---|---|
| Qué es | otro nombre para el mismo archivo | un archivo que contiene una ruta |
| En `ls -l` | mismo inodo, contador sube a 2 | tipo `l`, `nombre -> destino` |
| Si borrás el original | el contenido sigue | queda roto |
| Límites | mismo disco, no directorios | cualquier cosa |

```bash
rm demo/x.txt
cat demo/x-duro.txt
cat demo/x-link.txt
```

---

## Globbing

| Patrón | Coincide con |
|---|---|
| `*` | cualquier cosa |
| `?` | un carácter |
| `[abc]` `[0-9]` | uno de esos |
| `{a,b}` `{1..5}` | alternativas / secuencia |

```bash
echo /etc/host*
ls /etc/*.conf | wc -l
touch demo/f{1..3}.txt
ls demo/f?.txt
echo demo/*
rm -r demo
cd
```

> `echo patrón` muestra qué va a recibir el comando. Hacelo antes de un `rm`.
