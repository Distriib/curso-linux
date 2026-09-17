# Día 5 — Redes, DNS y SSH

## Qué vamos a hacer

1. **Conceptos de red** — IP, máscara, gateway, DNS y puertos: los cinco números que hay que saber leer
2. **Inspeccionar** — `ip`, `ss`, `ping` y `dig`: qué red tiene el servidor, sin tocar nada
3. **Configurar IP fija** — una segunda tarjeta de red y un perfil de NetworkManager con `nmcli`
4. **SSH a fondo** — entrar con clave y sin contraseña, atajos, copiar archivos y un túnel
5. **Diagnóstico por capas** — un método ordenado para encontrar dónde se corta la red

Hasta hoy trabajamos **adentro** del servidor. Hoy el servidor empieza a hablar con el resto de la red: le damos una dirección fija, entramos por clave y aprendemos a encontrar por qué "no llega".

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 1 | 10 | **Comandos:** IP, máscara, gateway, DNS, puertos (explicado en consola) |
| 1 | 10 | **Lab:** calcular una red con `ipcalc` — yo hago el primero, ustedes los demás, foto |
| 2 | 5 | **Comandos:** `ip`, `ss`, `ping`, `dig` (explicado en consola) |
| 2 | 20 | **Lab 1:** inspección de la VM — yo hago el primero, ustedes los demás, foto |
| 2 | 10 | **Lab 2:** resolución de nombres — yo hago el primero, ustedes los demás, foto |
| 3 | 10 | **Comandos:** NetworkManager y `nmcli` (explicado en consola) |
| 3 | 15 | **Lab 1:** segundo adaptador de red — yo hago el primero, ustedes los demás, foto |
| 3 | 15 | **Lab 2:** IP fija con el perfil `lab` — yo hago el primero, ustedes los demás, foto |
| 3 | 20 | **Lab 3:** DNS, dos IPs, ruta y nombre del servidor — yo hago el primero, ustedes los demás, foto |
| — | 15 | Descanso |
| 4 | 10 | **Comandos:** SSH (explicado en consola) |
| 4 | 15 | **Lab 1:** claves SSH — yo hago el primero, ustedes los demás, foto |
| 4 | 10 | **Lab 2:** atajos y huella del servidor — yo hago el primero, ustedes los demás, foto |
| 4 | 15 | **Lab 3:** copiar archivos y un túnel — yo hago el primero, ustedes los demás, foto |
| 5 | 10 | **Comandos:** diagnóstico por capas (explicado en consola) |
| 5 | 20 | **Lab:** encontrar la falla capa por capa — yo hago el primero, ustedes los demás, foto |
| — | 0 | Reto individual — solo si sobra tiempo; si no, es tarea |
| — | 10 | Cierre y snapshot |
| — | 20 | Colchón (margen para imprevistos) |

---

## Al terminar

Dejar el servidor con una IP fija que sobrevive al reinicio, entrar por SSH con clave y sin contraseña, y encontrar dónde se corta la red recorriendo las capas en orden.

---

## Tarea

1. **Snapshot `dia05-fin`** con la VM apagada. Antes de apagar, volver al nombre corto: `sudo hostnamectl set-hostname rhel01`. **No borrar** el perfil `lab`, las claves ni los nombres de `/etc/hosts`: se usan los próximos días.
2. **Agregar dos discos de 5 GB** para mañana, con la VM apagada y **después** del snapshot: en VirtualBox, *Settings > Storage > Controller: SATA > Add hard disk > Create > VDI > Dynamically allocated > 5 GB*, dos veces. Al encender, `lsblk` tiene que listar `sdb` y `sdc` de `5G`. No tocarlos.
3. **El reto** (Ticket #RED-05) si no se hizo en clase. Mandar la tabla y la verificación por el chat.
