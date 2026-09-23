# Lab — Encontrar la falla capa por capa

Recorrer el método sobre una falla provocada, y ver el tráfico con `tcpdump`.

---

## Parte 1 — Qué hizo NetworkManager hoy

```bash
journalctl -u NetworkManager -n 15 --no-pager
```

**Comprobar:** líneas con hora, y frases como `state change:` y `Activation: successful, device activated`. Ahí está registrado cada `nmcli connection up` que hicieron hoy.

---

## Parte 2 — Provocar una falla y recorrer las capas

Vamos a romper algo a propósito:

```bash
sudo nmcli connection down lab
```

Ahora, **sin arreglarlo todavía**, recorrer las capas en orden:

```bash
nmcli device status
ip -br a
ip route
```

**Comprobar:**

| Capa | Qué se ve |
|---|---|
| 1 — enlace | `enp0s8 ethernet disconnected --` ← **acá falla** |
| 2 — IP | `enp0s8` sin dirección IPv4 |
| 3 — ruta | desapareció la línea de `192.168.56.0/24` |

**La falla está en la capa 1.** No hace falta seguir revisando DNS ni servicios: con la tarjeta caída, todo lo de arriba iba a fallar igual.

Arreglar:

```bash
sudo nmcli connection up lab
ip -br a show enp0s8
```

**Comprobar:** vuelve la IP.

---

## Parte 3 — Todo lo que NetworkManager sabe de una tarjeta

```bash
nmcli device show enp0s8
```

**Comprobar:** entre las líneas aparecen `GENERAL.STATE: 100 (connected)`, `GENERAL.CONNECTION: lab`, `WIRED-PROPERTIES.CARRIER: on` y las IPs.

Si dijera `IP4-CONNECTIVITY: limited` en esta tarjeta, es normal: tiene IP pero no sale a internet, porque la red host-only no lleva a ningún lado.

---

## Parte 4 — Ver el tráfico con `tcpdump`

Hace falta **una segunda ventana**. En la primera sesión de la VM:

```bash
sudo tcpdump -i enp0s8 -n icmp
```

Queda esperando. Desde **su computadora**:

```bash
ping -c 4 192.168.56.10
```
(En Windows: `ping -n 4 192.168.56.10`)

**Comprobar:** en la ventana del `tcpdump` aparecen las preguntas y las respuestas, de a pares:
```
IP 192.168.56.1 > 192.168.56.10: ICMP echo request, id 18, seq 1, length 64
IP 192.168.56.10 > 192.168.56.1: ICMP echo reply,   id 18, seq 1, length 64
```

Cortar con `Ctrl+C`.

**Lo que enseña:**
- Si vieran solo `request` y ninguna `reply`, el servidor **recibe pero no contesta** — firewall local.
- Si no vieran nada, el paquete **nunca llegó** — el problema está antes del servidor.

---

## Parte 5 — Ver una consulta de DNS pasar

En la primera ventana:

```bash
sudo tcpdump -i any -n udp port 53 -c 2
```

En la segunda:

```bash
dig +short redhat.com
```

**Comprobar:** en la primera ventana se ven dos líneas — la pregunta saliendo hacia el DNS y la respuesta volviendo:
```
IP 10.0.2.15.44120 > 10.0.2.3.53: ... A? redhat.com. (51)
IP 10.0.2.3.53 > 10.0.2.15.44120: ... A 34.235.198.240 (55)
```

El `-c 2` hace que se detenga solo después de dos paquetes.

---

# Solución — todos los comandos

```bash
# Parte 1 — qué hizo NetworkManager
journalctl -u NetworkManager -n 15 --no-pager

# Parte 2 — provocar la falla y recorrer las capas
sudo nmcli connection down lab
nmcli device status
ip -br a
ip route
sudo nmcli connection up lab
ip -br a show enp0s8

# Parte 3 — todo lo que NM sabe de la tarjeta
nmcli device show enp0s8
```

## Parte 4 — hacen falta dos ventanas
En la primera (VM):
```bash
sudo tcpdump -i enp0s8 -n icmp
```
En su propia computadora:
```bash
ping -c 4 192.168.56.10
```
`Ctrl+C` en la primera para cortar.

## Parte 5 — también con dos ventanas
En la primera:
```bash
sudo tcpdump -i any -n udp port 53 -c 2
```
En la segunda:
```bash
dig +short redhat.com
```
