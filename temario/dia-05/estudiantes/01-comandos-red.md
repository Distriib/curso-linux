# 1 — Conceptos de red: los cinco números

## IP y máscara

```
192.168.56.10/24
└─────┬─────┘└┬┘
   la IP      la máscara en corto (CIDR): cuántos bits son "la red"
```

| Escritura | Máscara larga | Red de `192.168.56.10` | Direcciones para máquinas |
|---|---|---|---|
| `/24` | `255.255.255.0` | `192.168.56.0` | `.1` a `.254` (254) |
| `/16` | `255.255.0.0` | `192.168.0.0` | 65.534 |
| `/26` | `255.255.255.192` | depende del cuarto número | 62 |

- La **red** es la primera dirección del rango (`.0`); el **broadcast** es la última (`.255`). Ninguna de las dos se le da a una máquina.
- Dos equipos se hablan **directo** solo si están en la misma red. Si no, necesitan un intermediario: el **gateway**.

## Gateway y ruta por defecto

```bash
ip route
```
```
default via 10.0.2.2 dev enp0s3 ...
10.0.2.0/24 dev enp0s3 ...
```

- `default via 10.0.2.2` = "todo lo que no sé a dónde va, se lo entrego a `10.0.2.2`". Ese es el gateway.
- Sin ruta por defecto no hay internet. Con una equivocada, tampoco.

## DNS: de nombre a IP

```bash
cat /etc/hosts
cat /etc/resolv.conf
grep hosts: /etc/nsswitch.conf
```

| Orden | Dónde busca | Qué es |
|---|---|---|
| 1 `files` | `/etc/hosts` | la agenda local: líneas `IP nombre`. **Gana siempre** |
| 2 `dns` | `/etc/resolv.conf` → `nameserver 10.0.2.3` | el directorio público. **Lo escribe NetworkManager**: si lo editás a mano, lo perdés |
| 3 `myhostname` | — | el propio nombre del servidor |

## Puertos: qué servicio dentro de la máquina

La IP identifica la máquina; el **puerto** (0 a 65535) identifica el servicio.

| Puerto | Servicio | Puerto | Servicio |
|---|---|---|---|
| 22 | SSH | 53 | DNS |
| 80 | HTTP | 2049 | NFS |
| 443 | HTTPS | 445 | SMB |

- **TCP**: con conexión, confiable (SSH, web). **UDP**: sin conexión, rápido (DNS, hora).
- Un **socket** es `IP:puerto`. `ss -tulpn` muestra quién escucha en cuál.

## IPv6 en dos líneas

- Toda tarjeta activa tiene una dirección `fe80::...` (*link-local*): solo sirve dentro del mismo cable. Se ve en `ip a` como `inet6 fe80::.../64 scope link`.
- Se configura igual que IPv4, con `nmcli`.

## Las tres redes del hipervisor

| Modo | IP de la VM | ¿Sale a internet? | ¿Tu PC llega a la VM? |
|---|---|---|---|
| **NAT** | `10.0.2.15/24`, gateway `10.0.2.2`, DNS `10.0.2.3` (igual en todas las VMs) | Sí | Solo con reenvío de puertos (`2222→22`) |
| **Host-only** | `192.168.56.x/24`; tu PC es `192.168.56.1` | No (no hay gateway) | Sí, directo por IP |
| **Bridged** | una IP de la red real de la oficina | Sí | Sí, y cualquier otra PC también |

Hoy: **NAT + Host-only**. NAT da internet; Host-only da una IP fija a la que se entra directo, sin `-p 2222`.

## Calcular una red

```bash
ipcalc -bmn 192.168.56.10/24
```
`-b` broadcast · `-m` máscara · `-n` red.
