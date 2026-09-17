# Comandos — bucles y funciones

## `for`: lo mismo, para cada cosa de una lista

```
for svc in sshd crond chronyd; do        ← svc toma cada valor, uno por vez
    echo "$svc: $(systemctl is-active $svc)"
done                                      ← cierra el bucle
```

| La lista puede ser | Ejemplo |
|---|---|
| palabras | `for svc in sshd crond chronyd` |
| números | `for i in {1..5}` → 1 2 3 4 5 |
| archivos | `for f in ~/empresa/logs/*.log` → cada archivo que termine en `.log` |
| la salida de un comando | `for u in $(cut -d: -f1 /etc/passwd)` |

En una sola línea: `for i in {1..3}; do echo "web0$i"; done`

## `while read`: una línea por vez de un archivo

```
while read -r fecha hora nivel resto; do   ← cada línea se reparte en estas variables
    echo "$hora $nivel -> $resto"           ← la última variable se lleva todo lo que sobra
done < app.log                              ← el archivo que se lee
```

Si el archivo tiene los campos separados por comas: `while IFS=, read -r usuario grupo comentario`.

## Contar cosas de un archivo

| Comando | Qué hace |
|---|---|
| `grep -c ERROR app.log` | cuántas líneas tienen `ERROR` |
| `cut -d' ' -f3 app.log \| sort \| uniq -c` | cuántas veces aparece cada valor del campo 3 |
| `wc -l < app.log` | cuántas líneas (sin imprimir el nombre del archivo) |

## Funciones: un bloque con nombre

```
log() {                                ← se define arriba…
    echo "$(date '+%F %T') $1"         ← $1 es lo que le pasen a la función
    logger -t backup "$1"
}

log "OK: respaldo creado"              ← …y se usa como un comando más
```

## Valores por defecto

| Escribir | Qué hace |
|---|---|
| `ORIGEN="${1:-/home/student/empresa}"` | usa `$1`; si no llegó, usa `/home/student/empresa` |
| `basename /home/student/empresa` | `empresa` (la última parte) |
| `dirname /home/student/empresa` | `/home/student` (todo menos la última parte) |

```bash
basename /home/student/empresa
dirname /home/student/empresa
```

## Lo que usa `backup.sh`

| Comando | Qué hace |
|---|---|
| `tar -czf ARCHIVO.tar.gz -C /home/student empresa` | comprime `empresa` parándose en `/home/student` (adentro queda `empresa/...`) |
| `tar -tzf ARCHIVO.tar.gz` | lista qué hay adentro |
| `tar -xzf ARCHIVO.tar.gz -C /tmp/restauracion` | extrae en esa carpeta |
| `find /datos/backups -name "empresa-*.tar.gz" -mtime +7 -print -delete` | busca los de más de 7 días, los muestra y los borra |
| `touch -d "-10 days" archivo` | crea el archivo con fecha de hace 10 días |
| `ls -t` | ordena por fecha, el más nuevo primero |
