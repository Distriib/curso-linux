# Lab — Calcular una red con `ipcalc` (comandos)

## Parte 1 — Las herramientas del día
```bash
sudo dnf install -y ipcalc bind-utils rsync tcpdump
```
Mientras descarga, decir qué es cada uno: `ipcalc` calcula redes, `bind-utils` trae `dig` (preguntar al DNS), `rsync` copia archivos, `tcpdump` mira el tráfico. Si a alguien le falla con `This system is not registered`: `sudo subscription-manager register` y repetir.

## Parte 2 — La carpeta del Día 2
```bash
ls ~/empresa ~/empresa/documentos
```
Si a alguien le falta:
```bash
mkdir -p ~/empresa/documentos ~/empresa/clientes ~/empresa/backups ~/empresa/logs
touch ~/empresa/documentos/informe1.txt
```

## Parte 3 — La red de hoy
```bash
ipcalc -bmn 192.168.56.10/24
ipcalc -bmn 10.0.2.15/24
```
Respuesta: red `192.168.56.0`, última dirección (broadcast) `192.168.56.255`. En tu UTM la red host-only es otra; mostrá igual `192.168.56.10/24`, que es la que ellos van a usar.

## Parte 4 — Una máscara que no termina en 0
```bash
ipcalc -bmn 172.16.40.130/26
ipcalc -bmn 172.16.40.200/26
```
Respuesta: no. `.130` está en `172.16.40.128`; `.200` está en `172.16.40.192`. Decir: "con `/26` la red se corta cada 64 direcciones: 0, 64, 128, 192. Dos máquinas en redes distintas no se hablan sin gateway, aunque los tres primeros números sean iguales."
