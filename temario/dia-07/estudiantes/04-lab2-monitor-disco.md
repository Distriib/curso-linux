# Lab 4.2 — Alerta de uso de disco

Vamos a escribir `monitor-disco.sh`: revisa el porcentaje de uso de cada sistema de archivos, avisa por el log del sistema cuando alguno pasa el umbral, y anota todo en `/var/log/monitor-disco.log`. Va en `/usr/local/bin` porque en el Bloque 5 lo va a ejecutar systemd. Donde dice **Ahora ustedes**, el comando no está escrito: hay que resolverlo y mandar foto. La solución está al final de la hoja.

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
/archivos    6%
/datos       8%
/stratis     1%
```
La segunda vez, sin la línea `Mounted on Use%`.

**Los números de cada VM son distintos, y la cantidad de líneas también:** salen todos los sistemas de archivos que quedaron montados del Día 6 (`/archivos`, `/datos`, `/stratis`) más `/backups` si hicieron el reto de ayer. `/stratis` siempre da un porcentaje bajísimo porque se presenta con 1 TiB virtual.

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
Revisados 5 sistemas de archivos, 0 alerta(s)
código: 0
Revisados 5 sistemas de archivos, 4 alerta(s)
código: 1
```
**El "Revisados N" y la cantidad de alertas de cada uno van a ser distintos** — dependen de qué quedó montado del Día 6 y de cuán llena está cada VM. Lo que tiene que pasar en todos es lo mismo: con umbral 80 **cero** alertas y código `0`; con umbral 5, **varias** alertas y código `1`. Con umbral 5 no alerta lo que esté al 5% o menos: la prueba es `-gt`, mayor estricto.

---

## Parte 5 — El rastro

**¿Dónde quedó anotado cada aviso?**

```bash
tail -4 /var/log/monitor-disco.log
sudo journalctl -t monitor-disco -n 2 --no-pager
```

**Comprobar:**
```
2026-09-16 09:30:02 OK: /stratis al 1%
2026-09-16 09:30:40 ALERTA: / al 18% (umbral 5%)
2026-09-16 09:30:40 ALERTA: /boot al 25% (umbral 5%)
2026-09-16 09:30:40 ALERTA: /datos al 8% (umbral 5%)
Sep 16 09:30:40 rhel01 monitor-disco[5820]: ALERTA: / al 18% (umbral 5%)
Sep 16 09:30:40 rhel01 monitor-disco[5822]: ALERTA: /boot al 25% (umbral 5%)
```
Las líneas de cada uno son distintas. Lo que importa: en `/var/log/monitor-disco.log` está **todo** (los `OK` también), y en el journal solo las **alertas**, porque el `logger` está dentro del `if`.

---

# Solución — todos los comandos

```bash
# Parte 1 — lo que el script va a leer
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs
df --output=target,pcent -x tmpfs -x devtmpfs -x efivarfs | tail -n +2

# Parte 2 — el script (se escribe en /tmp y después se instala)
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

# Parte 3 — instalarlo
sudo cp /tmp/monitor-disco.sh /usr/local/bin/monitor-disco.sh
sudo chmod 755 /usr/local/bin/monitor-disco.sh
ls -l /usr/local/bin/monitor-disco.sh

# Parte 4 — ejecutarlo con dos umbrales
sudo /usr/local/bin/monitor-disco.sh; echo "código: $?"
sudo /usr/local/bin/monitor-disco.sh 5; echo "código: $?"
```

**Parte 4 (Ahora ustedes)** — el tercer umbral:
```bash
sudo /usr/local/bin/monitor-disco.sh 20; echo "código: $?"
```
Alertan solo los que estén por **encima** de 20. En la mayoría de las VM es solo `/boot`; si `/` está bastante lleno, también. El número no es el mismo para todos.

```bash
# Parte 5 — el rastro
tail -4 /var/log/monitor-disco.log
sudo journalctl -t monitor-disco -n 2 --no-pager
```
El script se escribe en `/tmp` y se **copia** con `cp` a `/usr/local/bin`: así toma la etiqueta de seguridad de la carpeta destino. Con `mv` se lleva la de `/tmp` y systemd después no lo puede ejecutar.
