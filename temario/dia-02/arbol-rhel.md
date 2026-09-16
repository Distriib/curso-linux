# El árbol de RHEL 9

Hoja de referencia. Para tener abierta durante el Día 2 y para entregarles.

---

## La idea de fondo

En Windows cada disco tiene su letra: `C:`, `D:`. En Linux **no hay letras**.
Todo cuelga de un único punto de partida llamado **raíz**, que se escribe `/`.

Si conectás un disco nuevo, no aparece como `E:`. Lo enganchás en algún lugar
del árbol, por ejemplo en `/datos`, y a partir de ahí es una carpeta más. Eso
se llama **montar** y es el tema del Día 6.

Por eso una ruta siempre empieza con `/`. Es la dirección desde la raíz.

```
/etc/ssh/sshd_config
│  │   │
│  │   └─ el archivo
│  └───── carpeta ssh, dentro de etc
└──────── la raíz: el punto de partida
```

---

## Las carpetas que importan

| Carpeta | Qué guarda | Cuándo la van a usar |
|---|---|---|
| `/etc` | **Toda la configuración** del sistema y de los servicios | Todos los días del curso |
| `/var/log` | Los **registros**: qué pasó y cuándo | Día 4 y Día 10 |
| `/home` | La carpeta personal de cada usuario | Día 3 |
| `/root` | La carpeta personal de root. **No** es la raíz | Día 3 |
| `/usr/bin` | Los **programas** instalados | Día 4 |
| `/tmp` | Archivos temporales. Se borra al reiniciar | Día 3 y Día 7 |
| `/srv` | Datos que el servidor **sirve** a otros | Día 3 y Día 9 |
| `/opt` | Software de terceros que no viene de los repositorios | Mención |
| `/dev` | Los **dispositivos**: discos, terminales | Día 6 |
| `/proc` | El estado del sistema en vivo. No son archivos reales | Día 4 |
| `/boot` | Lo necesario para arrancar: kernel y GRUB | Día 10 |
| `/mnt` y `/media` | Puntos de montaje temporales y para USB | Día 6 |

---

## Las tres que más van a pisar

**`/etc` — la configuración.** Si un servicio se comporta de cierta manera, la
respuesta está acá. Es texto plano: se lee, se edita, se versiona.

```bash
ls /etc | head -20
cat /etc/os-release
cat /etc/hosts
```

**`/var/log` — qué pasó.** Cuando algo falla, la respuesta está acá. Es lo
primero que mira un administrador.

```bash
sudo ls /var/log
sudo tail -20 /var/log/secure
```

**`/home` — la gente.** Cada usuario tiene la suya y no puede entrar a la de
los demás. Lo comprueban en el Día 3.

```bash
ls /home
```

---

## Dos confusiones frecuentes

**`/root` no es la raíz.** La raíz es `/`. `/root` es la carpeta personal del
usuario root, igual que `/home/student` lo es de student. Se parecen en el
nombre y en nada más.

**`/proc` no contiene archivos de verdad.** Son ventanas al estado del kernel,
generadas en el momento. Si leés `/proc/meminfo` estás preguntándole al kernel
cuánta memoria hay ahora mismo, no abriendo un archivo del disco.

```bash
cat /proc/meminfo | head -5
ls -l /proc/meminfo      # dice 0 bytes, pero tiene contenido
```

Ese contraste funciona muy bien en clase.

---

## Por qué está separado así

No es capricho: es una norma llamada **FHS**, que siguen todas las
distribuciones. Por eso un administrador que viene de Ubuntu se orienta en RHEL
sin problema.

La lógica es agrupar por **qué se hace con cada cosa**:

- Lo que **se configura** → `/etc`
- Lo que **cambia solo** mientras el sistema corre (logs, colas, cachés) → `/var`
- Lo que **se instala** y no cambia → `/usr`
- Lo que **es de las personas** → `/home`

Eso tiene una consecuencia práctica que van a agradecer: para respaldar un
servidor, con `/etc`, `/home` y `/var` tenés casi todo. `/usr` se reinstala
desde los repositorios.

---

## Para verlo en vivo

```bash
ls -l /
tree -L 1 /
man 7 hier        # el manual oficial del árbol
```

`man 7 hier` es un buen cierre: describe todo esto, viene con el sistema, y
demuestra que no necesitan internet para consultarlo.
