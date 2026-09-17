# Lab — Booleanos (comandos)

Cinco minutos. Vos primero, ellos después, foto.

## Parte 1 — Cuántos interruptores tiene Apache
```bash
getsebool -a | grep -c httpd
getsebool httpd_can_network_connect httpd_enable_homedirs
```

## Parte 2 — Verlo con su descripción
```bash
sudo semanage boolean -l | grep httpd_can_network_connect
```
Salen tres líneas (`httpd_can_network_connect`, `_cobbler`, `_db`): la primera es la nuestra. `(off , off)` = (ahora, al arrancar).

## Parte 3 — Prenderlo persistente
```bash
sudo setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect
sudo semanage boolean -l -C
```
El `-P` tarda de 5 a 20 segundos porque recompila la política: que nadie lo corte.

## Parte 4 — Apagarlo
```bash
sudo setsebool -P httpd_can_network_connect off
getsebool httpd_can_network_connect
```
Qué decir: "hoy no hay proxy inverso. Un interruptor prendido 'por si acaso' es un permiso de más."
