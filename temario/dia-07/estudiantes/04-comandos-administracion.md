# Comandos — scripts de administración

Dos scripts que juntan todo lo de la mañana: uno crea usuarios leyendo un archivo, otro vigila el disco. No traen nada nuevo de Bash; traen comandos de administración.

## Altas de usuarios (`crear-usuarios.sh`)

| Comando | Qué hace |
|---|---|
| `[[ $EUID -ne 0 ]]` | verdadero si **no** soy root (`EUID` = mi número de usuario efectivo) |
| `getent group desarrollo` | ¿existe el grupo? (código `0` sí, `2` no) |
| `groupadd desarrollo` | crear el grupo |
| `id lmartinez` | ¿existe el usuario? |
| `useradd -m -G desarrollo -c "Luis Martinez" lmartinez` | crear el usuario con su carpeta, en ese grupo suplementario, con descripción |
| `echo "lmartinez:Cambiar.2026" \| chpasswd` | ponerle contraseña sin que pregunte |
| `chage -d 0 lmartinez` | obligarlo a cambiarla la primera vez que entre |
| `continue` | dentro de un bucle: saltar a la siguiente línea |

Con `sudo` hay que dar la **ruta completa** del script: `sudo ~/bin/crear-usuarios.sh`. `sudo` no busca en `~/bin`.

## Alerta de disco (`monitor-disco.sh`)

```bash
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs
```

| Parte | Qué hace |
|---|---|
| `--output=target,pcent` | solo dos columnas: punto de montaje y porcentaje usado |
| `-x tmpfs -x devtmpfs -x efivarfs` | sin los sistemas de archivos que viven en memoria |
| `\| tail -n +2` | desde la línea 2: sin el encabezado |
| `tr -d '%'` | borrar el signo `%` para que quede un número |
| `mktemp` | crea un archivo temporal con nombre único y dice cuál |
| `logger -p local0.warning -t monitor-disco "..."` | mensaje al log del sistema, con prioridad de aviso |

## Dónde va cada script

| Carpeta | Quién lo ejecuta | Cómo se instala |
|---|---|---|
| `~/bin` | vos | `cat > ~/bin/x.sh`, `chmod +x` |
| `/usr/local/bin` | root, `cron` del sistema, systemd | `sudo cp /tmp/x.sh /usr/local/bin/`, `sudo chmod 755` |

Se **copia** con `cp`, no se mueve con `mv`: así el archivo toma la etiqueta de seguridad de la carpeta destino y systemd puede ejecutarlo.
