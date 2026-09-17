# Comandos — un script que decide

## Códigos de salida

| Código | Convención |
|---|---|
| `0` | salió bien |
| `1` | mal uso (falta un argumento, falta `sudo`) |
| `2` en adelante | errores propios del script (la ruta no existe…) |

`exit 2` termina el script con ese código. `cron` y systemd miran ese número para saber si la tarea falló.

## Anatomía de un `if`

```
if [[ -d "$RUTA" ]]; then          ← si la prueba da verdadero…
    echo "es una carpeta"
elif [[ -f "$RUTA" ]]; then        ← si no, probar esta otra
    echo "es un archivo"
else                               ← si ninguna
    echo "no existe"
fi                                 ← cierra el if (es "if" al revés)
```

Los espacios dentro de `[[ ]]` son obligatorios.

## Pruebas

| Prueba | Verdadero si |
|---|---|
| `-e "$X"` | existe |
| `-f "$X"` | es un archivo |
| `-d "$X"` | es una carpeta |
| `-x "$X"` | es ejecutable |
| `-z "$X"` | la variable está vacía |
| `"$A" == "$B"` / `!=` | textos iguales / distintos |
| `$A -eq $B` | números iguales (`-ne` distintos, `-gt` mayor, `-lt` menor, `-ge`, `-le`) |
| `$# -ne 1` | no llegó exactamente un argumento |
| `! prueba` | lo contrario |

## Atajos

| Escribir | Qué hace |
|---|---|
| `a && b` | ejecuta `b` solo si `a` salió bien |
| `a \|\| b` | ejecuta `b` solo si `a` falló |
| `echo "ERROR: ..." >&2` | manda el mensaje a la salida de errores, no a la normal |
| `comando 2> /dev/null` | tira los errores a la basura |

## Que el script pare al primer error

```
set -e          ← segunda línea del script: si un comando falla, el script termina ahí
```

Sin `set -e`, el script sigue después del error y termina con código `0` como si nada.

## Revisar y depurar

| Comando | Qué hace |
|---|---|
| `bash -n script.sh` | revisa la sintaxis sin ejecutar nada |
| `bash -x script.sh` | ejecuta mostrando cada comando con `+` delante y las variables ya reemplazadas |

## Dejar rastro

```bash
logger -t revisar "Prueba desde el Bloque 2 por $USER"
sudo journalctl -t revisar -n 1 --no-pager
```

`logger` escribe en el log del sistema. Es la forma de saber qué hizo un script que corrió a las 3 de la mañana.
