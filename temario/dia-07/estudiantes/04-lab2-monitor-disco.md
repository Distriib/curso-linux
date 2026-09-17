# Lab 2 — Alerta de uso de disco

Vamos a escribir `monitor-disco.sh`: revisa el porcentaje de uso de cada sistema de archivos, avisa por el log del sistema cuando alguno pasa el umbral, y anota todo en `/var/log/monitor-disco.log`. Va en `/usr/local/bin` porque en el Bloque 5 lo va a ejecutar systemd.

| Dato | Valor |
|---|---|
| Umbral por defecto | `80` % |
| Log propio | `/var/log/monitor-disco.log` |
| Códigos | `0` sin alertas · `1` con alertas |

---

## Parte 1 — Lo que el script va a leer

**¿Cómo obtengo solo el punto de montaje y el porcentaje, sin encabezado?**

```bash
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs | tail -n +2
```

**Comprobar:**
```
Mounted on Use%
/           18%
/boot       25%
/datos       4%
```
La segunda vez, sin la línea `Mounted on Use%`. Los porcentajes de cada VM son distintos.

---

## Parte 2 — El script

**¿Cómo recorre esa salida y decide si avisar?**

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

**Comprobar:** `sintaxis ok`.

---

## Parte 3 — Instalarlo en `/usr/local/bin`

**¿Cómo se instala un script para que lo ejecute root o systemd?**

```bash
sudo cp /tmp/monitor-disco.sh /usr/local/bin/monitor-disco.sh
sudo chmod 755 /usr/local/bin/monitor-disco.sh
ls -l /usr/local/bin/monitor-disco.sh
```

**Comprobar:**
```
-rwxr-xr-x. 1 root root ... /usr/local/bin/monitor-disco.sh
```

---

## Parte 4 — Ejecutarlo con dos umbrales

**¿Qué cambia entre el umbral normal y uno absurdo?**

```bash
sudo /usr/local/bin/monitor-disco.sh; echo "código: $?"
sudo /usr/local/bin/monitor-disco.sh 5; echo "código: $?"
```

Ahora ustedes: con umbral `20`. ¿Cuántas alertas? Foto.

**Comprobar:**
```
Revisados 3 sistemas de archivos, 0 alerta(s)
código: 0
Revisados 3 sistemas de archivos, 2 alerta(s)
código: 1
```
Con umbral 5 alertan `/` y `/boot`, pero no `/datos` (4 no es mayor que 5). El número de "Revisados" depende de lo que quedó montado el Día 6.

---

## Parte 5 — El rastro

**¿Dónde quedó anotado cada aviso?**

```bash
tail -4 /var/log/monitor-disco.log
sudo journalctl -t monitor-disco -n 2 --no-pager
```

**Comprobar:**
```
2026-09-16 09:30:02 OK: /datos al 4%
2026-09-16 09:30:40 ALERTA: / al 18% (umbral 5%)
2026-09-16 09:30:40 ALERTA: /boot al 25% (umbral 5%)
2026-09-16 09:30:40 OK: /datos al 4%
Sep 16 09:30:40 rhel01 monitor-disco[5820]: ALERTA: / al 18% (umbral 5%)
Sep 16 09:30:40 rhel01 monitor-disco[5822]: ALERTA: /boot al 25% (umbral 5%)
```
