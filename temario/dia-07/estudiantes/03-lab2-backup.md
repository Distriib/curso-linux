# Lab 3.2 — `backup.sh` con retención

Vamos a escribir el script de respaldo del servidor: comprime una carpeta con la fecha en el nombre, borra los respaldos de más de 7 días y deja rastro en el log del sistema. Donde dice **Ahora ustedes**, el comando no está escrito: hay que resolverlo y mandar foto. La solución está al final de la hoja.

| Dato | Valor |
|---|---|
| Origen por defecto | `/home/student/empresa` |
| Destino | `/datos/backups` |
| Nombre del archivo | `empresa-AAAA-MM-DD-HHMM.tar.gz` |
| Retención | 7 días |
| Códigos | `0` ok · `1` el origen no existe |

---

## Parte 1 — Escribir el script

**¿Cómo se arma un script con valores por defecto, validación, una función y retención?**

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

**Comprobar:** `sintaxis ok`.

---

## Parte 2 — Ejecutarlo

**¿Qué pasa con los valores por defecto, y qué pasa si el origen no existe?**

```bash
backup.sh; echo "código: $?"
backup.sh /nada; echo "código: $?"
ls -lh /datos/backups
```

Ahora ustedes: respalden `/home/student/bin` con `backup.sh /home/student/bin` y miren `ls /datos/backups`. Foto.

**Comprobar:**
```
2026-09-16 10:58:10 OK: creado /datos/backups/empresa-2026-09-16-1058.tar.gz
2026-09-16 10:58:10 Retención: 0 respaldo(s) de más de 7 días borrado(s)
código: 0
2026-09-16 10:58:21 ERROR: el origen /nada no existe
código: 1
-rw-rw-r--. 1 student student ... empresa-2026-09-16-1058.tar.gz
```

---

## Parte 3 — La retención

**¿Cómo pruebo que borra los viejos si todavía no pasaron 7 días?**

```bash
touch -d "-10 days" /datos/backups/empresa-2026-08-24-2330.tar.gz
touch -d "-3 days" /datos/backups/empresa-2026-08-31-2330.tar.gz
backup.sh
ls /datos/backups
```

**Comprobar:**
```
... OK: creado /datos/backups/empresa-2026-09-16-1101.tar.gz
... Retención: 1 respaldo(s) de más de 7 días borrado(s)
bin-...tar.gz  empresa-2026-08-31-2330.tar.gz  empresa-2026-09-16-1058.tar.gz  empresa-2026-09-16-1101.tar.gz
```
El de 10 días desapareció. El de 3 días sigue.

---

## Parte 4 — Ver qué hay adentro y restaurar

**¿Cómo sé que el respaldo sirve?**

```bash
ULTIMO=$(ls -t /datos/backups/empresa-*.tar.gz | head -1)
echo "$ULTIMO"
tar -tzf "$ULTIMO" | head -5
mkdir -p /tmp/restauracion
tar -xzf "$ULTIMO" -C /tmp/restauracion
ls /tmp/restauracion/empresa
```

**Comprobar:**
```
/datos/backups/empresa-2026-09-16-1101.tar.gz
empresa/
empresa/backups/
...
backups  clientes  documentos  genera-log.sh  logs
```

---

## Parte 5 — El rastro

**¿Qué quedó en el log del sistema?**

```bash
sudo journalctl -t backup -n 2 --no-pager
```

**Comprobar:**
```
Sep 16 11:01:02 rhel01 backup[5210]: OK: creado /datos/backups/empresa-2026-09-16-1101.tar.gz
Sep 16 11:01:02 rhel01 backup[5212]: Retención: 1 respaldo(s) de más de 7 días borrado(s)
```

---

# Solución — todos los comandos

```bash
# Parte 1 — el script
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

# Parte 2 — ejecutarlo
backup.sh; echo "código: $?"
backup.sh /nada; echo "código: $?"
ls -lh /datos/backups
```

**Parte 2 (Ahora ustedes)** — respaldar otra carpeta pasándola como argumento:
```bash
backup.sh /home/student/bin
ls /datos/backups
```
Queda un `bin-FECHA.tar.gz`: el nombre lo saca de `basename "$ORIGEN"`, por eso cambia solo.

```bash
# Parte 3 — la retención
touch -d "-10 days" /datos/backups/empresa-2026-08-24-2330.tar.gz
touch -d "-3 days" /datos/backups/empresa-2026-08-31-2330.tar.gz
backup.sh
ls /datos/backups

# Parte 4 — ver adentro y restaurar
ULTIMO=$(ls -t /datos/backups/empresa-*.tar.gz | head -1)
echo "$ULTIMO"
tar -tzf "$ULTIMO" | head -5
mkdir -p /tmp/restauracion
tar -xzf "$ULTIMO" -C /tmp/restauracion
ls /tmp/restauracion/empresa

# Parte 5 — el rastro
sudo journalctl -t backup -n 2 --no-pager
```
El `find` solo borra los que empiezan con el mismo nombre (`empresa-*`): el `bin-*.tar.gz` no lo toca.
