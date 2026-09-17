# Lab — autofs: montar al usar (comandos)

## Parte 1 — Instalar y leer el mapa maestro
```bash
sudo dnf install -y autofs
grep auto.master.d /etc/auto.master
```

## Parte 2 — El mapa maestro del laboratorio
```bash
sudo vim /etc/auto.master.d/lab.autofs
```
```
/remoto     /etc/auto.remoto     --timeout=60
/-          /etc/auto.directo
```
```bash
cat /etc/auto.master.d/lab.autofs
```
Decir: "`/remoto` es una carpeta que autofs controla entera. `/-` quiere decir 'las rutas completas están en el otro archivo'."

## Parte 3 — Los dos mapas
```bash
sudo vim /etc/auto.remoto
```
```
compartido    -rw,sync    192.168.56.10:/srv/nfs/compartido
lectura       -ro         127.0.0.1:/srv/nfs/lectura
```
```bash
echo "/datos/nfs    -rw,sync    192.168.56.10:/srv/nfs/compartido" | sudo tee /etc/auto.directo
cat /etc/auto.remoto /etc/auto.directo
```
Pegar las líneas en el chat. Un error de tipeo acá es el motivo número uno de "no funciona".

## Parte 4 — Arrancar y mirar lo que creó
```bash
sudo systemctl enable --now autofs
systemctl is-active autofs
ls -ld /remoto /datos/nfs
ls /remoto
mount | grep autofs
```
La frase: "`ls /remoto` está vacío y está bien. autofs no muestra lo que nadie pidió."

## Parte 5 — Disparar los montajes
```bash
ls /remoto/compartido
mount | grep nfs4
cat /remoto/lectura/README.txt
cat /datos/nfs/prueba.txt
mount | grep nfs4
```
Decir: "nadie ejecutó `mount`. Entrar a la ruta lo hizo."

Si a alguien no le monta: `sudo automount -m` para ver si leyó el mapa, y `journalctl -u autofs --no-pager | tail -5`.

## Parte 6 — Se desmonta solo
```bash
cd ~
sleep 90
mount | grep nfs4
```
Mientras corre el `sleep`, explicar `/etc/autofs.conf`: `timeout = 300` es el valor por defecto (el que usa el mapa directo) y `browse_mode = no` es la razón de que `ls /remoto` salga vacío.

Si a alguien no se le desmontó: tiene una terminal parada dentro de `/remoto/compartido`, o no pasó todavía el ciclo (hasta 75 s). `cd ~` y esperar.

## Parte 7 — Cambios y diagnóstico
```bash
sudo automount -m | head -20
sudo systemctl reload autofs
journalctl -u autofs --no-pager | tail -3
```
Decir: "después de editar cualquier mapa: `reload`. Lo van a necesitar en el reto."
