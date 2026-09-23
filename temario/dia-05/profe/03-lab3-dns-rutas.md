# Lab — Modificar el perfil (comandos)

## Parte 1
```bash
sudo nmcli connection modify lab ipv4.dns 1.1.1.1 +ipv4.dns 8.8.8.8 ipv4.dns-search lab.local
cat /etc/resolv.conf
sudo nmcli connection up lab
cat /etc/resolv.conf
```

## Parte 2
```bash
sudo nmcli connection modify lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"
sudo nmcli connection up lab
ip -br a show enp0s8
```
Desde el equipo del participante: `ping 192.168.56.11`

## Parte 3
```bash
sudo nmcli connection modify lab +ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route
sudo nmcli connection modify lab -ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route
```

## Parte 4
```bash
sudo hostnamectl set-hostname rhel01.lab.local
hostnamectl --static
hostname
echo "192.168.56.10 rhel01.lab.local rhel01" | sudo tee -a /etc/hosts
echo "192.168.56.10 servidor-nfs servidor-smb" | sudo tee -a /etc/hosts
getent hosts servidor-nfs
ping -c 1 rhel01.lab.local
getent hosts servidor-smb
```

## Parte 5
```bash
nmcli connection show lab | grep ipv4
```

---

## Qué señalar

- **Parte 1:** es **la** demostración del día. Que miren `resolv.conf` **antes** del `up` y vean que no cambió. Frase: *"acá está el error número uno: modificaron, no pasó nada, y creen que el comando no sirve. Falta el `up`"*.
  Detalle si preguntan: los DNS nuevos se usan para salir a internet por la tarjeta del NAT, no por la host-only. El DNS es del sistema entero, no de una tarjeta.
- **Parte 2:** dos IPs en una tarjeta. Se usa para publicar dos servicios con IPs distintas en un mismo servidor. El `ping` a la `.11` desde el PC lo comprueba.
- **Parte 3:** `proto static` en `ip route` = la puso una persona y es persistente. Comparar con una ruta puesta a mano con `ip route add`, que desaparece al reiniciar.
  **Es importante quitarla al final**: apunta a un gateway que no existe y en el reto puede hacer fallar la activación del perfil.
- **Parte 4:** avisar **antes** de que el prompt no va a cambiar. Si no, tres personas van a decir que no les funcionó. Se comprueba con `hostname`.
  Los alias `servidor-nfs` y `servidor-smb` los usan los Días 9 y 10: que no los borren.
- **Parte 5:** `grep ipv4` filtra la salida larga de `connection show`, que trae más de cien líneas.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| "Cambié el DNS y `resolv.conf` sigue igual" | Falta `up`. Es lo que el lab quiere provocar |
| "El prompt sigue diciendo `rhel01`" | Correcto; es solo el prompt |
| `nmcli con up lab` falla después de poner la ruta | El gateway de la ruta no existe. Quitarla con el `-` |
| Las dos IPs no aparecen | Escribieron la lista sin comillas, o con espacio después de la coma |
| Se les cortó la sesión | Estaban conectados por `192.168.56.10` en vez de por `-p 2222` |
