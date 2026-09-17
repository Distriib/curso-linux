# Lab — Un script que decide (comandos)

Pegar el bloque de `revisar.sh` y los dos de `set -e` en el chat.

## Parte 1 — Escribir el script
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
Si `bash -n` se queja: casi siempre falta un `fi`, un `then` o una comilla. Que copien el bloque de nuevo.

## Parte 2 — Probarlo con rutas reales
```bash
revisar.sh /etc
revisar.sh /etc/hostname
revisar.sh ~/bin/hola.sh
```
Ellos:
```bash
revisar.sh /home
revisar.sh /etc/passwd
revisar.sh /usr/bin/ls
```

## Parte 3 — Los errores y sus códigos
```bash
revisar.sh /nada; echo "código: $?"
revisar.sh; echo "código: $?"
revisar.sh /nada 2> /dev/null; echo "código: $?"
```

## Parte 4 — `set -e`: parar al primer error
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

## Parte 5 — Ver qué hace el script paso a paso
```bash
bash -x ~/bin/revisar.sh /etc/hostname
```
Ellos:
```bash
bash -x ~/bin/revisar.sh /nada
```

## Parte 6 — Dejar rastro en el log del sistema
```bash
logger -t revisar "Prueba de logger por $USER"
sudo journalctl -t revisar -n 1 --no-pager
```
