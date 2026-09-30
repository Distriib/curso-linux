# Práctica — Ticket #PGN-088

**Solos, 30 minutos.** Todo lo que hace falta ya lo vieron. Al final mandan **una sola foto** con el resultado de la verificación.

> Si se traban en algo, sáltenlo y sigan con lo siguiente. Al final de esta hoja hay una sección de ayuda.

---

## Tareas

### 1. El grupo y la persona

Crear el grupo **`archivo`** con número **3005**, y el usuario **`rvega`** con:

- número de usuario **2020**
- su carpeta personal
- descripción: `Raquel Vega - Archivo`
- que pertenezca al grupo `archivo` **además** de su grupo propio
- contraseña `Pgn.2026`

---

### 2. La carpeta de trabajo

Dentro de su propia carpeta (`~`), crear esta estructura, **con un solo comando**:

```
~/archivo-digital/
├── expedientes/
├── resoluciones/
└── respaldos/
```

---

### 3. Los documentos

Dentro de `expedientes/`, crear **cuatro** archivos:

| Archivo | Contenido |
|---|---|
| `exp-001.txt` | `Expediente 001 - APROBADO` |
| `exp-002.txt` | `Expediente 002 - PENDIENTE` |
| `exp-003.txt` | `Expediente 003 - APROBADO` |
| `exp-004.txt` | `Expediente 004 - RECHAZADO` |

---

### 4. Las consultas

Responder estas tres preguntas **con un comando cada una** (anoten el comando y el resultado):

1. ¿Cuántos expedientes están `APROBADO`?
2. ¿Cuáles son los nombres de los archivos que dicen `APROBADO`? (solo los nombres, sin el contenido)
3. ¿Cuántos archivos `.txt` hay en total dentro de `~/archivo-digital`, contando todas las subcarpetas?

---

### 5. El permiso

Dejar `exp-004.txt` de forma que **solo su dueño pueda leerlo y escribirlo**, y nadie más pueda hacer nada con él.

---

### 6. El respaldo

Empaquetar y comprimir toda la carpeta `~/archivo-digital` dentro de `~/archivo-digital/respaldos/`, con este nombre:

```
archivo-AAAA-MM-DD.tar.gz
```

donde `AAAA-MM-DD` es **la fecha de hoy, generada por el sistema**, no escrita a mano.

Después, **listar el contenido del paquete sin extraerlo**.

---

## Verificación — esto es lo que mandan al chat

Copien y peguen esta línea completa, y manden la foto de la salida:

```bash
id rvega; getent group archivo; tree ~/archivo-digital 2>/dev/null || ls -R ~/archivo-digital; ls -l ~/archivo-digital/expedientes/exp-004.txt; ls -lh ~/archivo-digital/respaldos/
```

---
---

# Ayuda — si te trabaste

## Qué comando usar en cada punto

| Punto | Comandos que necesitás |
|---|---|
| 1 | `groupadd`, `useradd`, `usermod`, `passwd` |
| 2 | `mkdir` con la opción que crea varias de una vez, y llaves `{}` |
| 3 | `echo` con `>` |
| 4 | `grep` (con las opciones de contar y de mostrar solo nombres) y `find` con `wc` |
| 5 | `chmod` |
| 6 | `tar` y `date` |

## Recordatorios

- Para que un grupo exista **antes** que el usuario que lo usa, se crea primero el grupo.
- Al agregar a alguien a un grupo, si se olvida la opción de "agregar", se **reemplazan** todos sus grupos.
- En `chmod`, `r=4`, `w=2`, `x=1`, sumados por terna: dueño, grupo, otros.
- En `tar`, la `f` va **última** de las letras, porque le sigue el nombre del archivo.
- `$(date +%F)` da la fecha de hoy como `2026-09-23`.

---
---

# Solución

**Solo miren esto cuando ya lo hayan intentado.**

### 1
```bash
sudo groupadd -g 3005 archivo
sudo useradd -m -u 2020 -c "Raquel Vega - Archivo" rvega
sudo usermod -aG archivo rvega
echo 'Pgn.2026' | sudo passwd --stdin rvega
id rvega
```

### 2
```bash
mkdir -p ~/archivo-digital/{expedientes,resoluciones,respaldos}
```

### 3
```bash
cd ~/archivo-digital/expedientes
echo "Expediente 001 - APROBADO" > exp-001.txt
echo "Expediente 002 - PENDIENTE" > exp-002.txt
echo "Expediente 003 - APROBADO" > exp-003.txt
echo "Expediente 004 - RECHAZADO" > exp-004.txt
```

### 4
```bash
grep -c APROBADO ~/archivo-digital/expedientes/*.txt
grep -l APROBADO ~/archivo-digital/expedientes/*.txt
find ~/archivo-digital -name "*.txt" | wc -l
```
La primera da una línea por archivo con su cuenta; para el total de expedientes aprobados, son **dos** (`exp-001` y `exp-003`). La segunda los nombra. La tercera da `4`.

### 5
```bash
chmod 600 ~/archivo-digital/expedientes/exp-004.txt
ls -l ~/archivo-digital/expedientes/exp-004.txt
```
Tiene que quedar `-rw-------`.

### 6
```bash
tar -czf ~/archivo-digital/respaldos/archivo-$(date +%F).tar.gz -C ~ archivo-digital
tar -tzf ~/archivo-digital/respaldos/archivo-$(date +%F).tar.gz
ls -lh ~/archivo-digital/respaldos/
```
