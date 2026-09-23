# 5 — Diagnóstico por capas

Un ticket nunca dice "el gateway está mal". Dice **"no funciona el correo"** o **"no puedo entrar al servidor"**.

El método: recorrer las capas **de abajo hacia arriba**, una pregunta por capa, y **detenerse en la primera que falla**. No se avanza hasta que la capa actual responde bien.

## Las siete preguntas, en orden

| # | Pregunta | Comando | Respuesta sana |
|---|---|---|---|
| 1 | ¿Hay enlace? | `ip link`, `nmcli device status` | `state UP`, `LOWER_UP`, `connected` |
| 2 | ¿Tengo la IP correcta? | `ip -br a` | la IP y la máscara esperadas, en la tarjeta esperada |
| 3 | ¿Llego al gateway? | `ip route`, `ping -c 3 GATEWAY` | `default via` correcto, 0% de pérdida |
| 4 | ¿Salgo a internet por IP? | `ping -c 3 8.8.8.8` | responde |
| 5 | ¿Resuelvo nombres? | `getent hosts redhat.com`, `dig @8.8.8.8 redhat.com` | responde |
| 6 | ¿El servicio escucha? | `sudo ss -tulpn` | `LISTEN` en el puerto esperado |
| 7 | ¿El firewall lo deja pasar? | `sudo firewall-cmd --list-all` | el servicio en la lista |

## De síntoma a capa

| Lo que reporta el usuario | Capa | Primer comando |
|---|---|---|
| "No tengo red, nada funciona" | 1–2 | `nmcli device status` |
| "Llego a los equipos de mi red pero no a internet" | 3 | `ip route` — ¿hay `default via`? |
| "`ping 8.8.8.8` funciona pero `ping google.com` no" | 5 | `cat /etc/resolv.conf` |
| "Resuelve lento y después funciona" | 5 | el primer `nameserver` no responde |
| "Desde mi PC no llego al servidor, pero él sí sale a internet" | 6–7 | `ss -tulpn` y `firewall-cmd --list-all` |
| "Funciona por IP pero no por nombre" | 5 | `/etc/hosts` |
| "Después de reiniciar perdí la IP fija" | 2 | la pusieron con `ip addr add` y no con `nmcli` |
| "Cambié el DNS en `resolv.conf` y volvió a lo de antes" | 5 | lo reescribió NetworkManager |

## Dos herramientas más

**Qué hizo NetworkManager y cuándo:**
```bash
journalctl -u NetworkManager -n 15 --no-pager
```
Ante un "se cayó la red a las 3 de la mañana", este log dice si fue el cable, el DHCP, o alguien que ejecutó un comando.

**¿Está llegando el tráfico o no?**
```bash
sudo tcpdump -i enp0s8 -n icmp
```
Es la pregunta definitiva:

- Si el paquete **no aparece**, el problema está **antes** del servidor (la red, el hipervisor, un firewall de por medio).
- Si **aparece y no hay respuesta**, el problema está **en** el servidor (el servicio, el firewall local, SELinux).
