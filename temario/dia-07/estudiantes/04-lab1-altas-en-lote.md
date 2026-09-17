# Lab 1 — Altas de usuarios en lote

Vamos a crear cuatro usuarios leyendo un archivo de texto. Contraseña inicial de todos: **`Cambiar.2026`**, y tienen que cambiarla al entrar.

| Usuario | Grupo | Nombre |
|---|---|---|
| `lmartinez` | `desarrollo` (nuevo) | Luis Martinez - Desarrollo |
| `rgomez` | `desarrollo` (nuevo) | Rosa Gomez - Desarrollo |
| `jperez` | `soporte` | Juan Perez - Soporte |
| `mcastro` | `auditoria` | Maria Castro - Auditoria |

---

## Parte 1 — El archivo de entrada

**¿Qué formato tiene que tener el archivo para que `read` lo reparta en variables?**

```bash
cat > ~/usuarios.txt <<'EOF'
lmartinez desarrollo Luis Martinez - Desarrollo
rgomez desarrollo Rosa Gomez - Desarrollo
jperez soporte Juan Perez - Soporte
mcastro auditoria Maria Castro - Auditoria
EOF
getent group soporte auditoria desarrollo
```

**Comprobar:**
```
soporte:x:3002:pedro
auditoria:x:3003:laura
```
`desarrollo` no aparece: no existe todavía. El script lo va a crear.

---

## Parte 2 — El script

**¿Cómo recorre el archivo y decide qué hacer con cada línea?**

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

**Comprobar:** `sintaxis ok`.

---

## Parte 3 — Las validaciones

**¿Qué pasa sin `sudo`, y qué pasa sin archivo?**

```bash
crear-usuarios.sh ~/usuarios.txt; echo "código: $?"
sudo ~/bin/crear-usuarios.sh; echo "código: $?"
```

**Comprobar:**
```
ERROR: ejecutar con sudo
código: 1
Uso: /home/student/bin/crear-usuarios.sh ARCHIVO
código: 1
```

---

## Parte 4 — Crear los usuarios

**¿Qué hizo con cada línea?**

```bash
sudo ~/bin/crear-usuarios.sh ~/usuarios.txt
id mcastro
sudo chage -l lmartinez | head -2
```

**Comprobar:**
```
Grupo desarrollo creado
Usuario lmartinez creado en grupo desarrollo
Usuario rgomez creado en grupo desarrollo
Usuario jperez creado en grupo soporte
Usuario mcastro creado en grupo auditoria
Resumen: 4 creado(s), 0 omitido(s)
uid=...(mcastro) gid=...(mcastro) groups=...(mcastro),3003(auditoria)
Last password change					: password must be changed
Password expires					: password must be changed
```

---

## Parte 5 — Correrlo dos veces no rompe nada

**¿Qué pasa si lo ejecuto otra vez con el mismo archivo?**

```bash
sudo ~/bin/crear-usuarios.sh ~/usuarios.txt
```

Ahora ustedes: agreguen una línea al archivo con un usuario inventado en el grupo `sistemas` (`echo "tsoto sistemas Tomas Soto - Sistemas" >> ~/usuarios.txt`) y vuelvan a ejecutar el script. Foto.

**Comprobar:**
```
Usuario lmartinez ya existe, se omite
Usuario rgomez ya existe, se omite
Usuario jperez ya existe, se omite
Usuario mcastro ya existe, se omite
Resumen: 0 creado(s), 4 omitido(s)
```
Con la línea nueva: `Resumen: 1 creado(s), 4 omitido(s)`.
