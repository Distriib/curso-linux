# 5 — Diagnóstico por capas (guía del instructor)

Diez minutos. Este bloque es un **método**, no comandos nuevos: todos los comandos ya los vieron hoy. Lo que se enseña es el orden.

**Qué es "por capas":** la red funciona por niveles apilados. Si el de abajo está roto, todos los de arriba fallan también. Por eso se revisa de abajo hacia arriba y se para en el primero que falla: arreglar ese suele arreglar todo lo demás.
**Qué es `tcpdump`:** un programa que muestra los paquetes que pasan por una tarjeta, en vivo. Responde la única pregunta que ninguna otra herramienta responde: *¿el tráfico está llegando o no?*
**Qué es `journalctl -u NetworkManager`:** el historial de lo que hizo NetworkManager, con hora. Se usa cuando la red se cayó y nadie estaba mirando.

---

## Lo que hay que transmitir

**El orden importa más que los comandos.** Frase para la clase: *"el error del principiante es empezar por el DNS porque es lo que más suena. Si el cable está desconectado, pueden revisar el DNS toda la tarde"*.

**Detenerse en la primera capa que falla.** No seguir bajando la lista: arreglar esa y volver a empezar desde arriba.

**La tabla de síntomas es la que van a usar de verdad.** Un ticket llega escrito como en la columna izquierda. Que aprendan a traducirlo a la columna del medio.

Dos que conviene señalar, porque ya les pasaron hoy:
- *"Después de reiniciar perdí la IP fija"* → la pusieron con `ip addr add` en vez de con `nmcli`. Hoy vieron por qué.
- *"Cambié el DNS en `resolv.conf` y volvió solo a lo de antes"* → lo reescribió NetworkManager en la siguiente activación. También lo vieron hoy.

**La regla de `tcpdump`,** que vale para toda la carrera:
- El paquete **no aparece** → el problema está **antes** del servidor.
- El paquete **aparece y no hay respuesta** → el problema está **en** el servidor.

Eso corta por la mitad cualquier discusión de "es la red" / "es el servidor".

## Advertencia sobre `tcpdump`

Si capturan el puerto 22 estando conectados por SSH, la propia salida genera más tráfico SSH y se retroalimenta sin parar. Por eso en el lab se usa `-c N` (parar después de N paquetes). Decirlo antes de que a alguien se le llene la pantalla.
