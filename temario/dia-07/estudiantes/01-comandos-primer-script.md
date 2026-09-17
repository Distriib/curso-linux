# Comandos — primer script

## Qué es un script

Una lista de comandos guardada en un archivo. Todo lo que funciona en la terminal funciona adentro.

```
#!/bin/bash                      ← línea 1: con qué programa se ejecuta (bash)
# hola.sh - qué hace este script ← comentario: el sistema lo ignora, la gente lo lee
NOMBRE="Procuraduria"            ← una variable
echo "Hola, $NOMBRE"             ← un comando, igual que en la terminal
```

## Escribirlo sin abrir un editor

```bash
cat > /tmp/prueba.sh <<'EOF'
#!/bin/bash
echo "Hola desde un script"
EOF
bash /tmp/prueba.sh
```

Todo lo que va entre `<<'EOF'` y la línea `EOF` se guarda en el archivo, tal cual.

## Tres formas de ejecutarlo

| Forma | Necesita | Cuándo |
|---|---|---|
| `bash hola.sh` | nada | para probar |
| `./hola.sh` | permiso `x` (`chmod +x hola.sh`) | estando en su carpeta |
| `hola.sh` | permiso `x` y estar en una carpeta del `PATH` | desde cualquier lugar |

```bash
echo "$PATH"
```

`~/bin` ya está en el `PATH` en RHEL 9. Scripts propios → `~/bin`. Scripts del servidor (los ejecuta root o systemd) → `/usr/local/bin`.

## Variables

| Escribir | Qué hace |
|---|---|
| `NOMBRE="Ana"` | asignar. **Sin espacios** alrededor del `=` |
| `$NOMBRE` | leer |
| `${NOMBRE}_reporte.txt` | leer, cuando sigue texto pegado |
| `"Hola $NOMBRE"` | comillas dobles: **sí** reemplaza la variable |
| `'Hola $NOMBRE'` | comillas simples: **no** reemplaza nada |
| `FECHA=$(date +%F)` | guardar la salida de un comando |
| `$((7 * 6))` | cuenta con enteros → `42` |
| `read -p "Nombre: " TECNICO` | pedir un dato por teclado y guardarlo |

Regla: las variables con rutas o texto van **siempre** entre comillas dobles: `"$RUTA"`.

## Lo que el script recibe

```
args.sh servidor01 "Sala de servidores" 42
  │         │              │            └ $3
  │         │              └ $2  (las comillas lo mantienen como un solo argumento)
  │         └ $1
  └ $0
```

| Variable | Qué tiene |
|---|---|
| `$0` | nombre del script |
| `$1` `$2` … | primer argumento, segundo… |
| `$#` | cuántos argumentos llegaron |
| `$@` | todos los argumentos |
| `$?` | código de salida del último comando: `0` = salió bien |

```bash
ls /etc/hostname; echo "código: $?"
ls /nada; echo "código: $?"
```
