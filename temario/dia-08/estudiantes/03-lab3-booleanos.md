# Lab 3.3 — Booleanos

Vamos a prender un interruptor de la política de forma persistente, verlo, y volverlo a apagar.

---

## Parte 1 — Cuántos interruptores tiene Apache

**¿Qué puede prenderse de Apache sin escribir reglas?**

```bash
getsebool -a | grep -c httpd
getsebool httpd_can_network_connect httpd_enable_homedirs
```

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo. Foto.

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

Ahora ustedes: lo mismo (el primero tarda unos segundos). Foto.

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

Ahora ustedes: lo mismo. Foto.

**Comprobar:** `httpd_can_network_connect --> off`. Hoy no hay proxy inverso; prender interruptores "por si acaso" agranda la superficie de ataque.
