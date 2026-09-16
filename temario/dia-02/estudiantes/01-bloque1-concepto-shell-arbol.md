# 1 — Cómo funciona la shell

## El prompt

```
[student@rhel01 ~]$
```

`student` = usuario · `rhel01` = equipo · `~` = directorio actual · `$` = usuario normal (`#` = root)

## Anatomía de un comando

```
comando [opciones] [argumentos]
ls   -l -a -h   /etc
ls   -lah        /etc
```

- Opciones cortas (`-l`) se combinan: `-lah`
- Opciones largas (`--human-readable`) no se combinan

## Rutas

| Ejemplo | Qué es |
|---|---|
| `/etc/hostname` | ruta absoluta: empieza con `/`, apunta siempre al mismo lugar sin importar dónde estés parado |
| `documentos/informe.txt` | ruta relativa: depende de en qué carpeta estás parado ahora |
| `.` | el directorio actual |
| `..` | el directorio padre (uno arriba) |
| `~` | tu home (`/home/student`) |
| `cd -` | volver al directorio anterior |

## Variables de entorno

| Variable | Qué es |
|---|---|
| `$HOME` | tu directorio home |
| `$PATH` | dónde busca los programas la shell |
| `$PS1` | el formato del prompt |
| `$?` | código de salida del último comando (`0` = éxito) |

```bash
echo $PATH
export MIVAR=hola
```

## Alias

```bash
alias lh='ls -lh'
type lh
```

## Historial

| Comando | Qué hace |
|---|---|
| `history` | listar lo escrito |
| `!!` | repetir el último comando |
| `!n` | repetir el comando número `n` |
| `!$` | último argumento del comando anterior |
| `Ctrl+R` | buscar hacia atrás en el historial |

## Atajos de edición

| Atajo | Efecto |
|---|---|
| `Ctrl+A` / `Ctrl+E` | ir al inicio / al final de la línea |
| `Ctrl+U` / `Ctrl+K` | borrar hasta el inicio / hasta el final |
| `Ctrl+W` | borrar la palabra anterior |
| `Ctrl+Y` | pegar lo borrado |
| `Ctrl+L` | limpiar pantalla |

## El árbol de RHEL 9

| Directorio | Qué guarda |
|---|---|
| `/etc` | configuración del sistema |
| `/var` / `/var/log` | datos variables / logs |
| `/usr` | programas instalados |
| `/home` | usuarios |
| `/root` | home de root |
| `/tmp` | temporales |
| `/opt` | software de terceros |
| `/srv` | datos servidos |
| `/dev` | dispositivos |
| `/proc` / `/sys` | virtuales del kernel |
| `/boot` | kernel y GRUB |
| `/mnt` / `/media` | montajes |
| `/run` | estado en ejecución (RAM) |
