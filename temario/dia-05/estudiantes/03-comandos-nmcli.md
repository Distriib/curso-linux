# 3 — Configurar la red con NetworkManager

**NetworkManager es el único que maneja la red en RHEL 9.** Los archivos viejos de RHEL 7 (`ifcfg-*`) ya no se usan.

## Dónde se guarda la configuración

```bash
sudo ls -l /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/enp0s3.nmconnection
```

Un archivo por perfil, de root, permisos `600`:

```
[connection]
id=enp0s3
type=ethernet
interface-name=enp0s3

[ipv4]
method=auto

[ipv6]
method=auto
```

Cada línea del archivo es una propiedad de `nmcli`: `[ipv4] method=auto` es lo mismo que `ipv4.method auto`.

## El ciclo: crear, modificar, aplicar

```bash
sudo nmcli connection add ...      # 1. crea el perfil (escribe el archivo)
sudo nmcli connection modify ...   # 2. cambia el archivo — NO aplica nada todavía
sudo nmcli connection up NOMBRE    # 3. aplica
ip -br a                           # 4. verifica
```

> **El error número uno del día: modificar y no aplicar.**
> `modify` es editar el archivo. `up` es reiniciar la tarjeta con lo que dice el archivo.

## Crear un perfil con IP fija

```bash
sudo nmcli connection add type ethernet con-name lab ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.56.10/24
```

| Parte | Qué dice |
|---|---|
| `type ethernet` | es una tarjeta de red común |
| `con-name lab` | el perfil se va a llamar `lab` |
| `ifname enp0s8` | se aplica a esa tarjeta |
| `ipv4.method manual` | IP fija (con `auto` sería por DHCP) |
| `ipv4.addresses ...` | la IP y la máscara |

## Propiedades que se usan hoy

| Propiedad | Para qué |
|---|---|
| `ipv4.method` | `auto` (DHCP) o `manual` (fija) |
| `ipv4.addresses` | una o varias IPs, separadas por coma |
| `ipv4.gateway` | por dónde salir |
| `ipv4.dns` | servidores DNS |
| `ipv4.dns-search` | dominio que se agrega a los nombres cortos |
| `ipv4.routes` | rutas hacia otras redes |
| `connection.autoconnect` | si se activa solo al arrancar |

El signo `+` delante agrega a una lista y el `-` quita:
```bash
sudo nmcli connection modify lab +ipv4.dns 8.8.8.8
sudo nmcli connection modify lab -ipv4.dns 8.8.8.8
```

## Comandos de consulta

```bash
nmcli device status          # tarjetas y qué perfil tienen puesto
nmcli connection show        # perfiles que existen
nmcli connection show lab    # todo lo que dice un perfil
```

## Otras formas de hacer lo mismo

| Herramienta | Cuándo |
|---|---|
| `nmcli` | siempre: se copia y pega en un ticket, y es lo que pide el examen |
| `nmtui` | menús de texto, para quien recién empieza |
| Editar el `.nmconnection` | para copiar una configuración entre servidores. Después: `nmcli connection reload` |

## Nombre del servidor y nombres locales

```bash
sudo hostnamectl set-hostname rhel01.lab.local
hostnamectl --static
echo "192.168.56.10 servidor-nfs" | sudo tee -a /etc/hosts
getent hosts servidor-nfs
```
