# Lab 3.3 — Booleanos

Vamos a prender un interruptor de la política de forma persistente, verlo, y volverlo a apagar.

---

## Parte 1 — Cuántos interruptores tiene Apache

**¿Qué puede prenderse de Apache sin escribir reglas?**

```bash
getsebool -a | grep -c httpd
getsebool httpd_can_network_connect httpd_enable_homedirs
```

**Comprobar:**
```
45
httpd_can_network_connect --> off
httpd_enable_homedirs --> off
```
El número puede variar un poco. Los dos que importan hoy están apagados.

---

## Parte 2 — Verlo con su descripción

**¿Qué hace `httpd_can_network_connect`?**

```bash
sudo semanage boolean -l | grep httpd_can_network_connect
```

**Comprobar:**
```
httpd_can_network_connect      (off  ,  off)  Allow httpd to can network connect
...
```
Los dos valores son `(ahora, al arrancar)`.

---

## Parte 3 — Prenderlo persistente

**¿Cómo lo prendo para que sobreviva al reinicio?**

```bash
sudo setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect
sudo semanage boolean -l -C
```

(El primer comando tarda unos segundos.)

**Comprobar:**
```
httpd_can_network_connect --> on
SELinux boolean                State  Default Description
httpd_can_network_connect      (on   ,   on)  Allow httpd to can network connect
```

---

## Parte 4 — Apagarlo

**¿Por qué lo apagamos?**

```bash
sudo setsebool -P httpd_can_network_connect off
getsebool httpd_can_network_connect
```

**Comprobar:** `httpd_can_network_connect --> off`. Hoy no hay proxy inverso; prender interruptores "por si acaso" agranda la superficie de ataque.

---

# Solución — todos los comandos

```bash
# Parte 1 — cuántos interruptores tiene Apache
getsebool -a | grep -c httpd
getsebool httpd_can_network_connect httpd_enable_homedirs

# Parte 2 — verlo con su descripción
sudo semanage boolean -l | grep httpd_can_network_connect

# Parte 3 — prenderlo persistente (tarda unos segundos)
sudo setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect
sudo semanage boolean -l -C

# Parte 4 — apagarlo otra vez
sudo setsebool -P httpd_can_network_connect off
getsebool httpd_can_network_connect
```

**El `-P` es lo que importa.** Sin `-P` el cambio se pierde al reiniciar, y es el clásico "funcionaba y después del reinicio dejó de funcionar". Con `-P` se escribe en la política y sobrevive.

Los dos valores entre paréntesis de `semanage boolean -l` son `(ahora, al arrancar)`: si no coinciden, alguien lo cambió sin `-P`.

Un booleano es la respuesta correcta cuando el AVC dice que el servicio quiere hacer algo **legítimo pero apagado por defecto** — típicamente salir a la red (`{ name_connect }`) o leer los `home` de los usuarios. No es la respuesta para una etiqueta mal puesta.
