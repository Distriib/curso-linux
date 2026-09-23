# Lab — IP fija con el perfil `lab` (comandos)

## Parte 1
```bash
sudo ls -l /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/enp0s3.nmconnection
```

## Parte 2
```bash
sudo nmcli connection add type ethernet con-name lab ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.56.10/24
```

## Parte 3
```bash
sudo nmcli connection up lab
ip -br a
ip route
nmcli device status
```

## Parte 4
```bash
sudo nmcli connection delete "Wired connection 1"
nmcli connection show
sudo cat /etc/NetworkManager/system-connections/lab.nmconnection
```

## Parte 5
En el equipo del participante:
```bash
ping -c 2 192.168.56.10
ssh student@192.168.56.10
hostname
exit
```

## Parte 6
```bash
sudo nmcli connection down lab
sudo nmcli connection up lab
ip -br a show enp0s8
```

---

## Qué señalar

- **Parte 2:** leer el comando de izquierda a derecha nombrando cada pieza. Y decir por qué **no** lleva gateway: *"esta red no va a ningún lado. El gateway ya lo tienen en la otra tarjeta"*.
- **Parte 3:** acá se ve por fin device ≠ connection: tarjeta `enp0s8`, perfil `lab`. Y la ruta nueva a `192.168.56.0/24` **sin** ser `default`.
- **Parte 5:** es el momento del día. Entran a la VM por su IP, como a un servidor real. Funciona de entrada porque el firewall de RHEL ya permite `ssh`. Desde esta IP se van a probar NFS y Samba más adelante.
- **Parte 6:** la prueba de que es persistente. Frase: *"si hubieran puesto la IP con `ip addr add`, al bajar y subir la tarjeta se habría perdido. Con `nmcli` está escrita en un archivo"*.

## Cuidado

Los labs de este bloque se hacen **desde la sesión del NAT** (`ssh -p 2222`). Si alguien está conectado a `192.168.56.10` y hace `nmcli con up lab`, se corta a sí mismo. Decirlo antes de la Parte 6, que es donde pasa.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `Error: unknown connection 'Wired connection 1'` | No existía. Seguir |
| La IP no aparece después del `add` | Falta el `up`. Es el error del día |
| `Error: Connection activation failed` | El nombre de la tarjeta en `ifname` no es el suyo. Comprobar con `nmcli device status` |
| Desde el PC no responde el `ping` | La red host-only del hipervisor no está creada, o eligieron otra IP que no es del rango |
| Se les cortó la sesión | Estaban conectados por la IP nueva en vez de por `-p 2222` |
| En UTM la IP no funciona | El rango lo elige macOS: tiene que ser del rango que muestra `ifconfig` en el Mac |
