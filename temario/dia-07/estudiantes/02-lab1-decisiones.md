# Lab — Un script que decide

Vamos a escribir `revisar.sh`: recibe una ruta y dice si es una carpeta, un archivo o nada, con un código de salida distinto en cada caso.

| Caso | Mensaje | Código |
|---|---|---|
| sin argumento | `Uso: ... RUTA` | `1` |
| carpeta | `... es una carpeta con N elementos` | `0` |
| archivo | `... es un archivo de N bytes` | `0` |
| no existe | `ERROR: ... no existe` | `2` |

---

## Parte 1 — Escribir el script

**¿Cómo valida un script lo que recibe antes de trabajar?**

```bash
cat > ~/bin/revisar.sh <<'EOF'
#!/bin/bash
# revisar.sh - dice qué es una ruta: carpeta, archivo o nada
# Uso:    revisar.sh RUTA
# Salida: 0 ok · 1 falta el argumento · 2 la ruta no existe
if [[ $# -ne 1 ]]; then
    echo "Uso: $0 RUTA" >&2
    exit 1
fi

RUTA="$1"

if [[ -d "$RUTA" ]]; then
    echo "$RUTA es una carpeta con $(ls -A "$RUTA" | wc -l) elementos"
elif [[ -f "$RUTA" ]]; then
    echo "$RUTA es un archivo de $(stat -c %s "$RUTA") bytes"
    [[ -x "$RUTA" ]] && echo "  y es ejecutable"
else
    echo "ERROR: $RUTA no existe" >&2
    exit 2
fi
EOF
chmod +x ~/bin/revisar.sh
bash -n ~/bin/revisar.sh && echo "sintaxis ok"
```

**Comprobar:** `sintaxis ok`.

---

## Parte 2 — Probarlo con rutas reales

**¿Qué camino toma el `if` con cada ruta?**

```bash
revisar.sh /etc
revisar.sh /etc/hostname
revisar.sh ~/bin/hola.sh
```

Ahora ustedes: con `/home`, `/etc/passwd` y `/usr/bin/ls`. Foto.

**Comprobar:**
```
/etc es una carpeta con ... elementos
/etc/hostname es un archivo de 7 bytes
/home/student/bin/hola.sh es un archivo de ... bytes
  y es ejecutable
```

---

## Parte 3 — Los errores y sus códigos

**¿Qué pasa cuando la ruta no existe, o cuando no le paso nada?**

```bash
revisar.sh /nada; echo "código: $?"
revisar.sh; echo "código: $?"
revisar.sh /nada 2> /dev/null; echo "código: $?"
```

**Comprobar:**
```
ERROR: /nada no existe
código: 2
Uso: /home/student/bin/revisar.sh RUTA
código: 1
código: 2
```
En el tercero el mensaje desapareció: iba por la salida de errores y la tiramos. El código quedó.

---

## Parte 4 — `set -e`: parar al primer error

**¿Qué hace un script cuando un comando del medio falla?**

```bash
cat > /tmp/sin-set.sh <<'EOF'
#!/bin/bash
echo "Antes del error"
cat /noexiste
echo "Después del error (se ejecuta igual)"
EOF
cat > /tmp/con-set.sh <<'EOF'
#!/bin/bash
set -e
echo "Antes del error"
cat /noexiste
echo "Después del error (NO se ejecuta)"
EOF
bash /tmp/sin-set.sh; echo "código: $?"
bash /tmp/con-set.sh; echo "código: $?"
```

**Comprobar:**
```
Antes del error
cat: /noexiste: No such file or directory
Después del error (se ejecuta igual)
código: 0
Antes del error
cat: /noexiste: No such file or directory
código: 1
```

---

## Parte 5 — Ver qué hace el script paso a paso

**¿Cómo veo qué valor tenía cada variable cuando el script hizo algo raro?**

```bash
bash -x ~/bin/revisar.sh /etc/hostname
```

Ahora ustedes: `bash -x ~/bin/revisar.sh /nada`. Foto.

**Comprobar:**
```
+ [[ 1 -ne 1 ]]
+ RUTA=/etc/hostname
+ [[ -d /etc/hostname ]]
+ [[ -f /etc/hostname ]]
++ stat -c %s /etc/hostname
+ echo '/etc/hostname es un archivo de 7 bytes'
/etc/hostname es un archivo de 7 bytes
+ [[ -x /etc/hostname ]]
```
Cada `+` es un comando ya con sus variables reemplazadas. `++` es uno que está dentro de `$( )`.

---

## Parte 6 — Dejar rastro en el log del sistema

**¿Cómo se entera alguien de lo que hizo un script que nadie estaba mirando?**

```bash
logger -t revisar "Prueba de logger por $USER"
sudo journalctl -t revisar -n 1 --no-pager
```

**Comprobar:**
```
Sep 16 ... rhel01 revisar[...]: Prueba de logger por student
```
