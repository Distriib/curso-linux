# Lab — Diagnóstico por capas (comandos)

## Parte 1
```bash
journalctl -u NetworkManager -n 15 --no-pager
```

## Parte 2
```bash
sudo nmcli connection down lab
nmcli device status
ip -br a
ip route
sudo nmcli connection up lab
ip -br a show enp0s8
```

## Parte 3
```bash
nmcli device show enp0s8
```

## Parte 4
Primera ventana (VM):
```bash
sudo tcpdump -i enp0s8 -n icmp
```
Equipo del participante:
```bash
ping -c 4 192.168.56.10
```
`Ctrl+C` para cortar.

## Parte 5
Primera ventana:
```bash
sudo tcpdump -i any -n udp port 53 -c 2
```
Segunda ventana:
```bash
dig +short redhat.com
```

---

## Qué señalar

- **Parte 1:** cada `up` y `down` de hoy dejó rastro con hora. Frase: *"cuando les digan 'la red se cayó de madrugada', esto les dice si fue el cable, el DHCP, o una persona"*.
- **Parte 2:** es el corazón del bloque. Que recorran las tres capas **antes** de arreglar, y que vean que la respuesta ya estaba en la primera. Frase: *"encontraron la falla en el primer comando. Si hubieran empezado por el DNS, seguirían buscando"*.
  Detalle: la tarjeta sigue `UP` a nivel de enlace aunque el perfil esté caído — NetworkManager la deja levantada para detectar el cable. Lo que falta es la IP.
- **Parte 4:** hace falta una segunda sesión SSH abierta. Decirlo antes. Lo que tienen que ver son los **pares** request/reply.
  Las dos conclusiones de la regla (no aparece = antes del servidor; aparece sin respuesta = en el servidor) valen para toda la carrera. Vale la pena repetirlas.
- **Parte 5:** el `-c 2` es obligatorio. Sin él, capturar puerto 53 desde una sesión SSH llena la pantalla.
  Si el primer `nameserver` de su `resolv.conf` es `1.1.1.1` en vez de `10.0.2.3`, van a ver esa IP. Está bien.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `tcpdump: command not found` | Se instaló en el primer lab del día |
| No aparece nada en la captura ICMP | Están mirando la tarjeta equivocada; tiene que ser la host-only |
| La pantalla se llena sin parar | Capturaron el puerto 22 sin `-c`. `Ctrl+C` |
| `tcpdump` no arranca en segundo plano | No usar `&`: si `sudo` pide contraseña, queda colgado |
| Después del `down`, se cortó la sesión | Estaban conectados por `192.168.56.10`. Entrar por `-p 2222` |
