# Reto — Ticket #RED-05 (solución)

## Bloque para pegar en el chat

Que lo corran **desde la sesión `ssh -p 2222`**. Las dos fallas son seguras: no cortan esa sesión.

```bash
sudo nmcli connection modify lab ipv4.addresses 192.168.57.10/24
sudo nmcli connection up lab
sudo nmcli connection modify enp0s3 ipv4.ignore-auto-dns yes ipv4.dns 192.0.2.53
sudo nmcli connection up enp0s3
echo "Listo. A diagnosticar."
```

**Qué rompe cada una:**

1. **IP equivocada en `lab`** — la tarjeta host-only queda en `192.168.57.10`, otra red. Desde el equipo del participante, `ssh rhel01` deja de llegar.
2. **DNS inexistente** — `192.0.2.53` no existe (es un rango reservado para documentación). El servidor sale a internet por IP pero no resuelve nombres. `ignore-auto-dns yes` hace que ignore el DNS que le da el DHCP.

**Antes de la clase:** correrlo en tu VM y comprobar que tu sesión `-p 2222` sobrevive. Después, restaurar con la solución de abajo.

## Solución

| Falla | Síntoma | Comando que la delata | Corrección |
|---|---|---|---|
| IP de `lab` | Desde el PC, `ssh rhel01` no llega. Internet funciona | `ip -br a` → `enp0s8 ... 192.168.57.10/24` | `sudo nmcli connection modify lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"` y `sudo nmcli connection up lab` |
| DNS | `ping 8.8.8.8` funciona; `ping redhat.com` falla con `Temporary failure in name resolution`; `dig @8.8.8.8 redhat.com` sí funciona | `cat /etc/resolv.conf` → `nameserver 192.0.2.53` | `sudo nmcli connection modify enp0s3 ipv4.ignore-auto-dns no ipv4.dns ""` y `sudo nmcli connection up enp0s3` |

Verificación final:
```bash
ip -br a show enp0s8
cat /etc/resolv.conf
ping -c 2 8.8.8.8
getent hosts redhat.com
```
Y desde el equipo del participante: `ssh rhel01 hostname`

## Qué se evalúa

1. **Recorrió las capas en orden** y se detuvo en la correcta. La falla del DNS solo se ve en la capa 5; la de la IP, en la 2.
2. **Corrigió con `nmcli`**, no con comandos volátiles.
3. **Verificó desde fuera** de la VM, no solo adentro.
4. La tabla nombra **el comando** que delató cada falla, no solo la falla.

## Si alguien se enreda

Restaurar a mano:
```bash
sudo nmcli connection modify lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"
sudo nmcli connection up lab
sudo nmcli connection modify enp0s3 ipv4.ignore-auto-dns no ipv4.dns ""
sudo nmcli connection up enp0s3
```
