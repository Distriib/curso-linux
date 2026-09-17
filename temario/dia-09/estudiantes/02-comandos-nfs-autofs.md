# Comandos — NFS y autofs

## NFS: el disco de red del mundo Linux

El **servidor exporta** una carpeta; el **cliente la monta** y la usa como si fuera local. RHEL 9 usa NFS versión 4.2: un solo puerto, `2049/tcp`.

| Lado | Paquete | Servicio |
|---|---|---|
| servidor | `nfs-utils` | `nfs-server` |
| cliente | `nfs-utils` | ninguno: solo `mount` |

## `/etc/exports` — quién puede montar qué

```
/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)
   │                        │          └ opciones, SIN espacio antes del paréntesis
   │                        └ quién puede montar: una IP o una red
   └ qué carpeta se exporta
```

| Opción | Qué hace |
|---|---|
| `rw` / `ro` | lectura y escritura / solo lectura |
| `sync` | el servidor confirma la escritura cuando llegó al disco |
| `root_squash` | el root del cliente se convierte en `nobody` en el servidor (por defecto) |
| `no_root_squash` | el root del cliente es root también acá (cómodo en laboratorio, peligroso en producción) |

Un espacio entre la red y el paréntesis exporta **a todo el mundo** con opciones por defecto.

| Comando | Qué hace |
|---|---|
| `sudo exportfs -rav` | aplicar lo que dice `/etc/exports` (`r` releer, `a` todo, `v` detallado) |
| `sudo exportfs -v` | qué está exportado, con todas sus opciones |
| `showmount -e servidor` | qué exporta un servidor |

Firewall: tres servicios, `nfs`, `rpc-bind` y `mountd`, en las dos zonas.

NFS identifica a las personas por **número**: el UID 1000 del cliente es el UID 1000 del servidor. Por eso `student` escribe desde el cliente y el archivo aparece como `student` en el servidor.

## Cliente: tres formas de montar

| Forma | Comando | Dura |
|---|---|---|
| a mano | `sudo mount -t nfs 192.168.56.10:/srv/nfs/compartido /mnt/nfs` | hasta el reinicio |
| permanente | una línea en `/etc/fstab` | siempre |
| a demanda | autofs | se monta al usarla, se suelta al dejar de usarla |

Línea de `fstab`:
```
192.168.56.10:/srv/nfs/compartido  /mnt/nfs  nfs  defaults,_netdev,nofail  0 0
                                                            │       └ si no puede montar, sigue arrancando
                                                            └ esperar a que haya red
```
Después de tocar `fstab`: `sudo systemctl daemon-reload` y `sudo mount -a` **antes** de reiniciar.

## autofs: montar cuando se usa

| Archivo | Qué es |
|---|---|
| `/etc/auto.master` | el mapa maestro original: no se toca |
| `/etc/auto.master.d/*.autofs` | nuestros mapas maestros: qué carpeta vigilar y dónde está el detalle |
| `/etc/auto.remoto` | un mapa **indirecto**: una carpeta padre con subcarpetas |
| `/etc/auto.directo` | un mapa **directo**: rutas completas |

Mapa maestro (`/etc/auto.master.d/lab.autofs`):
```
/remoto     /etc/auto.remoto     --timeout=60
/-          /etc/auto.directo
 │              │                     └ segundos sin uso hasta desmontar
 │              └ archivo con el detalle
 └ carpeta padre; "/-" = las rutas completas están en el mapa
```

Mapa indirecto (`/etc/auto.remoto`) — la primera columna es una **subcarpeta** de `/remoto`:
```
compartido    -rw,sync    192.168.56.10:/srv/nfs/compartido
lectura       -ro         127.0.0.1:/srv/nfs/lectura
```

Mapa directo (`/etc/auto.directo`) — la primera columna es una **ruta completa**:
```
/datos/nfs    -rw,sync    192.168.56.10:/srv/nfs/compartido
```

| Comando | Qué hace |
|---|---|
| `sudo systemctl enable --now autofs` | arrancar; autofs crea las carpetas solo |
| `ls /remoto` | sale **vacío**: las subcarpetas aparecen cuando se usan por su nombre exacto |
| `ls /remoto/compartido` | esto monta |
| `mount` y `df -hT` | qué está montado ahora (las líneas `nfs4`) |
| `sudo systemctl reload autofs` | releer los mapas después de editarlos |
| `sudo automount -m` | ver toda la configuración cargada |
