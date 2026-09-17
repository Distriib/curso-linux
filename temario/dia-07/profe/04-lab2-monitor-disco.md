# Lab 2 — Alerta de uso de disco (comandos)

Pegar el script en el chat.

## Parte 1 — Lo que el script va a leer
```bash
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs | tail -n +2
```
En VirtualBox salen `/`, `/boot` y `/datos` (3). En UTM también `/boot/efi` (4). Si además quedaron `/backups` o `/stratis` del Día 6, más. Todo vale.

## Parte 2 — El script
```bash
cat > /tmp/monitor-disco.sh <<'EOF'
#!/bin/bash
# monitor-disco.sh - avisa cuando un sistema de archivos supera el umbral de uso
# Uso:    monitor-disco.sh [UMBRAL]   (por defecto 80)
# Log:    /var/log/monitor-disco.log y el log del sistema (logger)
# Salida: 0 sin alertas · 1 con alertas
UMBRAL="${1:-80}"
LOG=/var/log/monitor-disco.log
FECHA=$(date '+%F %T')
alertas=0
revisados=0

TMP=$(mktemp)
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs | tail -n +2 > "$TMP"

while read -r punto uso; do
    uso=$(echo "$uso" | tr -d '%')
    revisados=$((revisados + 1))
    if [[ $uso -gt $UMBRAL ]]; then
        mensaje="ALERTA: $punto al ${uso}% (umbral ${UMBRAL}%)"
        logger -p local0.warning -t monitor-disco "$mensaje"
        alertas=$((alertas + 1))
    else
        mensaje="OK: $punto al ${uso}%"
    fi
    echo "$FECHA $mensaje" >> "$LOG"
done < "$TMP"
rm -f "$TMP"

echo "Revisados $revisados sistemas de archivos, $alertas alerta(s)"
[[ $alertas -eq 0 ]] && exit 0 || exit 1
EOF
bash -n /tmp/monitor-disco.sh && echo "sintaxis ok"
```

## Parte 3 — Instalarlo en `/usr/local/bin`
```bash
sudo cp /tmp/monitor-disco.sh /usr/local/bin/monitor-disco.sh
sudo chmod 755 /usr/local/bin/monitor-disco.sh
ls -l /usr/local/bin/monitor-disco.sh
```
`cp`, no `mv`. Si alguien lo movió: `sudo restorecon -v /usr/local/bin/monitor-disco.sh` le devuelve la etiqueta correcta.

## Parte 4 — Ejecutarlo con dos umbrales
```bash
sudo /usr/local/bin/monitor-disco.sh; echo "código: $?"
sudo /usr/local/bin/monitor-disco.sh 5; echo "código: $?"
```
Ellos:
```bash
sudo /usr/local/bin/monitor-disco.sh 20; echo "código: $?"
```
Ruta completa siempre: `sudo monitor-disco.sh` da `command not found` (`secure_path`).

## Parte 5 — El rastro
```bash
tail -4 /var/log/monitor-disco.log
sudo journalctl -t monitor-disco -n 2 --no-pager
```
