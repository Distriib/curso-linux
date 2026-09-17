# Lab 2 — `backup.sh` con retención (comandos)

Pegar el script en el chat.

## Parte 1 — Escribir el script
```bash
cat > ~/bin/backup.sh <<'EOF'
#!/bin/bash
# backup.sh - respalda una carpeta en tar.gz y borra los respaldos viejos
# Uso:    backup.sh [ORIGEN] [DESTINO]
# Salida: 0 ok · 1 el origen no existe
set -e

ORIGEN="${1:-/home/student/empresa}"
DESTINO="${2:-/datos/backups}"
RETENCION=7
FECHA=$(date +%F-%H%M)
NOMBRE=$(basename "$ORIGEN")
ARCHIVO="$DESTINO/${NOMBRE}-${FECHA}.tar.gz"

log() {
    echo "$(date '+%F %T') $1"
    logger -t backup "$1"
}

if [[ ! -d "$ORIGEN" ]]; then
    log "ERROR: el origen $ORIGEN no existe"
    exit 1
fi

mkdir -p "$DESTINO"
tar -czf "$ARCHIVO" -C "$(dirname "$ORIGEN")" "$NOMBRE"
log "OK: creado $ARCHIVO"

BORRADOS=$(find "$DESTINO" -name "${NOMBRE}-*.tar.gz" -mtime +$RETENCION -print -delete | wc -l)
log "Retención: $BORRADOS respaldo(s) de más de $RETENCION días borrado(s)"
EOF
chmod +x ~/bin/backup.sh
bash -n ~/bin/backup.sh && echo "sintaxis ok"
```
Leerlo con ellos: valores por defecto, la función `log`, la validación con `exit 1`, el `tar`, la retención con `find`.

## Parte 2 — Ejecutarlo
```bash
backup.sh; echo "código: $?"
backup.sh /nada; echo "código: $?"
ls -lh /datos/backups
```
Ellos:
```bash
backup.sh /home/student/bin
ls /datos/backups
```
Si `tar` da `Permission denied` en `/datos/backups`: faltó el `chown` de la Parte 1 del primer lab. `sudo chown student:student /datos/backups`.

## Parte 3 — La retención
```bash
touch -d "-10 days" /datos/backups/empresa-2026-08-24-2330.tar.gz
touch -d "-3 days" /datos/backups/empresa-2026-08-31-2330.tar.gz
backup.sh
ls /datos/backups
```

## Parte 4 — Ver qué hay adentro y restaurar
```bash
ULTIMO=$(ls -t /datos/backups/empresa-*.tar.gz | head -1)
echo "$ULTIMO"
tar -tzf "$ULTIMO" | head -5
mkdir -p /tmp/restauracion
tar -xzf "$ULTIMO" -C /tmp/restauracion
ls /tmp/restauracion/empresa
```
Decir: se usa `ls -t | head -1` y no el comodín directo porque los dos archivos de la Parte 3 están vacíos y no son `tar` válidos.

## Parte 5 — El rastro
```bash
sudo journalctl -t backup -n 2 --no-pager
```
