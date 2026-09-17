# Lab — Primer script (comandos)

Pegar cada bloque `cat > ... <<'EOF'` en el chat antes de que lo tipeen. Yo hago el primero, ellos los demás, foto.

## Parte 1 — Preparar las carpetas
```bash
mkdir -p ~/bin ~/empresa/logs
sudo mkdir -p /datos/backups
sudo chown student:student /datos/backups
echo "$PATH"
```
Si `/datos` no existe (Día 6 incompleto), `mkdir -p` lo crea como carpeta común. Sirve igual.

## Parte 2 — Escribir el primer script
```bash
cat > ~/bin/hola.sh <<'EOF'
#!/bin/bash
# hola.sh - primer script: variables, comillas y $( )
NOMBRE="Procuraduria"
FECHA=$(date +%F)

echo "Hola, $NOMBRE. Hoy es $FECHA"
echo 'Con comillas simples no se reemplaza: $NOMBRE'
echo "Archivo de salida: ${NOMBRE}_reporte.txt"
echo "Cuenta: $((7 * 6))"
EOF
ls -l ~/bin/hola.sh
bash ~/bin/hola.sh
```

## Parte 3 — Permiso de ejecución y `PATH`
```bash
cd ~/bin
./hola.sh
chmod +x hola.sh
./hola.sh
cd /tmp
hola.sh
cd
```
Si a alguien `hola.sh` le da `command not found` desde `/tmp`: que mire `ls -l ~/bin/hola.sh` (¿tiene la `x`?) y `echo "$PATH"` (¿está `/home/student/bin`?). Si no está: `source ~/.bashrc`.

## Parte 4 — Lo que recibe un script
```bash
cat > ~/bin/args.sh <<'EOF'
#!/bin/bash
# args.sh - muestra lo que recibe
echo "Nombre del script: $0"
echo "Primer argumento:  $1"
echo "Segundo argumento: $2"
echo "Cantidad:          $#"
echo "Todos:             $@"
EOF
chmod +x ~/bin/args.sh
args.sh servidor01 "Sala de servidores" 42
```
Ellos:
```bash
args.sh Ana "Dirección de Informática" 7
args.sh Ana Dirección de Informática 7
```
La segunda da `Cantidad: 5`.

## Parte 5 — El código de salida
```bash
ls /etc/hostname; echo "código: $?"
ls /nada; echo "código: $?"
```

## Parte 6 — Un reporte con datos del servidor
```bash
cat > ~/bin/reporte.sh <<'EOF'
#!/bin/bash
# reporte.sh - reporte breve del servidor
read -p "Nombre del técnico: " TECNICO
echo "=== Reporte de $(hostname) ==="
echo "Generado por: $TECNICO"
echo "Fecha:        $(date '+%F %T')"
echo "Kernel:       $(uname -r)"
echo "Encendido:    $(uptime -p)"
echo "Disco raíz:   $(df -h / | tail -1)"
EOF
chmod +x ~/bin/reporte.sh
reporte.sh
```
Ellos agregan con `vim ~/bin/reporte.sh` (`G` va al final, `o` abre una línea nueva, `Esc` `:wq`):
```bash
echo "Conectados:   $(who | wc -l)"
```
```bash
reporte.sh
```
