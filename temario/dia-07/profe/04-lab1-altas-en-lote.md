# Lab 1 — Altas de usuarios en lote (comandos)

Pegar el archivo de entrada y el script en el chat.

## Parte 1 — El archivo de entrada
```bash
cat > ~/usuarios.txt <<'EOF'
lmartinez desarrollo Luis Martinez - Desarrollo
rgomez desarrollo Rosa Gomez - Desarrollo
jperez soporte Juan Perez - Soporte
mcastro auditoria Maria Castro - Auditoria
EOF
getent group soporte auditoria desarrollo
```
Si `soporte` o `auditoria` no aparecen (Día 3 incompleto), el script los crea también. No pasa nada.

## Parte 2 — El script
```bash
cat > ~/bin/crear-usuarios.sh <<'EOF'
#!/bin/bash
# crear-usuarios.sh - crea usuarios en lote desde un archivo de texto
# Formato del archivo: usuario grupo nombre completo
# Uso:    sudo /home/student/bin/crear-usuarios.sh /home/student/usuarios.txt
# Salida: 0 ok · 1 error de uso
set -e

ARCHIVO="$1"
CLAVE_INICIAL="Cambiar.2026"

if [[ $EUID -ne 0 ]]; then
    echo "ERROR: ejecutar con sudo" >&2
    exit 1
fi
if [[ ! -f "$ARCHIVO" ]]; then
    echo "Uso: $0 ARCHIVO" >&2
    exit 1
fi

creados=0
omitidos=0

while read -r usuario grupo comentario; do
    if ! getent group "$grupo" > /dev/null; then
        groupadd "$grupo"
        echo "Grupo $grupo creado"
    fi

    if id "$usuario" > /dev/null 2>&1; then
        echo "Usuario $usuario ya existe, se omite"
        omitidos=$((omitidos + 1))
        continue
    fi

    useradd -m -G "$grupo" -c "$comentario" "$usuario"
    echo "$usuario:$CLAVE_INICIAL" | chpasswd
    chage -d 0 "$usuario"
    logger -t crear-usuarios "Usuario $usuario creado en grupo $grupo"
    echo "Usuario $usuario creado en grupo $grupo"
    creados=$((creados + 1))
done < "$ARCHIVO"

echo "Resumen: $creados creado(s), $omitidos omitido(s)"
EOF
chmod +x ~/bin/crear-usuarios.sh
bash -n ~/bin/crear-usuarios.sh && echo "sintaxis ok"
```
Leerlo con ellos de arriba a abajo: dos validaciones, un `while read`, adentro dos `if` y el `useradd`.

## Parte 3 — Las validaciones
```bash
crear-usuarios.sh ~/usuarios.txt; echo "código: $?"
sudo ~/bin/crear-usuarios.sh; echo "código: $?"
```

## Parte 4 — Crear los usuarios
```bash
sudo ~/bin/crear-usuarios.sh ~/usuarios.txt
id mcastro
sudo chage -l lmartinez | head -2
```
Los UID salen 2005 a 2008 si el Día 3 quedó como el material; si no, otros. No importa.

## Parte 5 — Correrlo dos veces no rompe nada
```bash
sudo ~/bin/crear-usuarios.sh ~/usuarios.txt
```
Ellos:
```bash
echo "tsoto sistemas Tomas Soto - Sistemas" >> ~/usuarios.txt
sudo ~/bin/crear-usuarios.sh ~/usuarios.txt
```
