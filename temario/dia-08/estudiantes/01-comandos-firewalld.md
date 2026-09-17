# Comandos — firewalld

## Por qué empezamos por el firewall

Cada puerto abierto es una puerta que un servicio mantiene abierta las 24 horas. RHEL recién instalado deja pasar tres cosas: `ssh`, `cockpit` (9090) y `dhcpv6-client`. Hoy decidimos nosotros qué queda abierto.

## Quién hace qué

```
firewall-cmd  →  firewalld (servicio)  →  nftables (kernel: el que filtra de verdad)

/etc/firewalld/          lo que configurás vos
/usr/lib/firewalld/      lo que viene de fábrica (no se edita)
```

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

## Zonas

Una zona es un nivel de confianza con su lista de lo permitido. Un paquete que llega se clasifica así: primero por la **red de origen** (`sources`), después por la **interfaz** por la que entró; si nada coincide, la zona por defecto (`public`). **El origen manda sobre la interfaz.**

| Zona | Qué deja pasar | Para qué |
|---|---|---|
| `drop` | nada; ni responde | redes hostiles |
| `block` | nada; responde "prohibido" | igual, pero avisa |
| `public` | solo lo listado: `ssh`, `cockpit`, `dhcpv6-client` — **zona por defecto** | interfaces expuestas |
| `internal` | un poco más: lo de `public` + `mdns`, `samba-client` | red interna de administración |
| `trusted` | **todo** | nunca, salvo una interfaz 100 % confiable |

## Servicios predefinidos

Un servicio de firewalld es un archivo que agrupa puertos bajo un nombre: `http` = `80/tcp`. Hay unos 190.

```bash
sudo firewall-cmd --get-services | wc -w
sudo firewall-cmd --info-service=http
cat /usr/lib/firewalld/services/http.xml
```

Si no hay un servicio para lo que querés abrir, se abre el puerto directo: `--add-port=82/tcp`.

## Runtime vs permanent — la regla de oro

| Comando | Se aplica ya | Sobrevive a `--reload` y al reinicio |
|---|---|---|
| `--add-service=http` | sí | **no** |
| `--permanent --add-service=http` | **no** (hasta `--reload`) | sí |

Dos formas de trabajar bien:
1. Probar sin `--permanent`; si funciona, `--runtime-to-permanent`.
2. Escribir con `--permanent` y aplicar con `--reload`.

El error número uno: `--permanent` sin `--reload` ("no funciona"), o sin `--permanent` y reiniciar ("dejó de funcionar").

## Anatomía

```
sudo firewall-cmd --permanent --zone=internal --add-service=http
                   │           │               └ qué: un servicio (o --add-port=82/tcp, o --add-source=RED)
                   │           └ en qué zona (sin --zone: la zona por defecto, public)
                   └ guardar en disco (sin esto, dura hasta el próximo --reload)
```

## Consultar

| Comando | Qué muestra |
|---|---|
| `--state` | si firewalld está corriendo |
| `--get-default-zone` | la zona por defecto |
| `--get-active-zones` | zonas que tienen interfaces u orígenes asignados |
| `--get-zones` | todas las zonas que existen |
| `--list-all` | todo lo de la zona por defecto |
| `--zone=internal --list-all` | todo lo de otra zona |
| `--info-zone=drop` | qué hace una zona |
| `--list-services` / `--list-ports` | solo servicios / solo puertos |
| `--permanent --list-ports` | lo guardado en disco |
| `--query-service=http` | `yes` / `no` |

## Cambiar

| Comando | Qué hace |
|---|---|
| `--add-service=http` / `--remove-service=http` | abrir / cerrar un servicio |
| `--add-port=82/tcp` / `--remove-port=82/tcp` | abrir / cerrar un puerto |
| `--zone=internal --add-source=192.168.56.0/24` | "lo que venga de esta red se evalúa en `internal`" |
| `--runtime-to-permanent` | guardar en disco lo que está corriendo |
| `--reload` | aplicar lo guardado (y borrar lo que no se guardó) |

Todo con `sudo firewall-cmd` adelante; con `--permanent` si tiene que durar.
