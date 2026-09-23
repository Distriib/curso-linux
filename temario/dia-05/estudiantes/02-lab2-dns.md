# Lab — Resolución de nombres

Preguntarle al DNS a mano, y ver la diferencia entre lo que responde el DNS y lo que usa el sistema.

---

## Parte 1 — Preguntar por un nombre

**¿Qué IP tiene `redhat.com`?**

```bash
dig redhat.com
dig +short redhat.com
```

Ahora ustedes: lo mismo con `google.com`. Foto.

**Comprobar:** en la salida larga, `status: NOERROR`, una `ANSWER SECTION` con un registro `A`, y al final `SERVER: 10.0.2.3#53` — **ese es el DNS que contestó**, el mismo de `/etc/resolv.conf`. La corta da solo la IP.

---

## Parte 2 — Preguntarle a otro DNS

**¿Y si el DNS de mi red está mal? ¿Cómo pregunto a otro?**

```bash
dig @8.8.8.8 +short redhat.com
```

Ahora ustedes: la consulta inversa, `dig -x 8.8.8.8 +short`. Foto.

**Comprobar:** la primera da la misma IP; la segunda dice `dns.google.`

Esto es la prueba definitiva cuando se sospecha del DNS: **si con `@8.8.8.8` responde y sin `@` no, el problema es el DNS de la red, no internet.**

---

## Parte 3 — Lo que ve el sistema, no solo el DNS

```bash
getent hosts localhost
getent hosts rhel01
getent hosts redhat.com
```

**Comprobar:** las tres responden, pero cada una viene de un lado distinto:

| Consulta | De dónde sale la respuesta |
|---|---|
| `localhost` | de `/etc/hosts` |
| `rhel01` | del propio nombre del servidor |
| `redhat.com` | del DNS |

`dig` pregunta **solo al DNS**. `getent` pregunta como lo haría cualquier programa del sistema. Por eso, cuando un servicio "no resuelve", se prueba con `getent`.

---

## Parte 4 — `/etc/hosts` le gana al DNS

**¿Qué pasa si pongo un nombre a mano en `/etc/hosts`?**

```bash
echo "127.0.0.1 redhat.com" | sudo tee -a /etc/hosts
getent hosts redhat.com
dig +short redhat.com
```

**Comprobar:** `getent` dice `127.0.0.1` y `dig` sigue diciendo la IP real. El sistema le hizo caso al archivo, no al DNS.

Ahora hay que dejarlo como estaba. Abrir el archivo y borrar esa última línea:

```bash
sudo vim /etc/hosts
```
(`G` para ir al final, `dd` para borrar la línea, `:wq`)

```bash
getent hosts redhat.com
```

**Comprobar:** vuelve a decir la IP real.

Este truco sirve para probar una web antes de cambiar el DNS de verdad — y es también una forma clásica de secuestrar un nombre. Revisar `/etc/hosts` es parte de cualquier diagnóstico de DNS.

---

# Solución — todos los comandos

```bash
# Parte 1
dig redhat.com
dig +short redhat.com
dig google.com

# Parte 2
dig @8.8.8.8 +short redhat.com
dig -x 8.8.8.8 +short

# Parte 3
getent hosts localhost
getent hosts rhel01
getent hosts redhat.com

# Parte 4
echo "127.0.0.1 redhat.com" | sudo tee -a /etc/hosts
getent hosts redhat.com
dig +short redhat.com
sudo vim /etc/hosts
```
En vim: `G` (última línea), `dd` (borrarla), `:wq` (guardar y salir).
```bash
getent hosts redhat.com
```
