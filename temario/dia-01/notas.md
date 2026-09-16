# Notas

conectarme a el ssh:
ssh -p 2222 student@localhost


## Variantes de descarga de la ISO

| Variante | Qué es | ¿La querés? |
|---|---|---|
| **Binary DVD** (~10 GB) | El instalador completo, trae todos los paquetes adentro. Instala sin internet | ✅ Esta |
| **Boot ISO** (~1 GB) | Instalador mínimo, descarga todo por internet mientras instala | ❌ Más lento y depende de la red |
| **KVM Guest Image** (`.qcow2`) | Máquina ya instalada, para servidores Linux. No es un instalador | ❌ |
| **Source DVD** | El código fuente. Para desarrolladores de paquetes | ❌ |

---

## Reglas de port forwarding

**SSH**

```
Protocol:       TCP
Guest Address:  (dejalo vacío)
Guest Port:     22
Host Address:   127.0.0.1
Host Port:      2222
```

**Web**

```
Protocol:       TCP
Guest Address:  (dejalo vacío)
Guest Port:     80
Host Address:   127.0.0.1
Host Port:      8080
```

---

## El prompt

```
[student@rhel01 ~]$
```

| Parte | Qué es |
|---|---|
| `student` | El usuario con el que entraste |
| `rhel01` | El hostname, el que pusiste en Network & Host Name |
| `~` | La carpeta donde estás parado: tu casa, `/home/student` |
| `$` | Usuario normal. Si fuera `#`, serías root |


dnf - gestor de paquetes de donde descargamos , la tienda pues
---

## Consola de la VM vs SSH

Son dos formas de llegar al mismo servidor.

**La consola** (la ventana de UTM o VirtualBox) es el monitor conectado
físicamente al servidor. Estás mirando la pantalla de la máquina.

**SSH** es conectarse por red desde otra computadora. Es como se administra
un servidor de verdad: nadie viaja al centro de datos a enchufar un monitor.

| | Consola | SSH |
|---|---|---|
| Copiar y pegar | ❌ | ✅ |
| Historial hacia arriba | ❌ | ✅ |
| Varias sesiones a la vez | ❌ | ✅ |
| Tamaño de letra, colores | Fijo | El de tu terminal |
| Funciona sin red | ✅ | ❌ |
| Funciona con el sistema roto | ✅ | ❌ |

### Cuándo se usa cada una

**SSH para todo**, porque es cómodo y es lo real.

**La consola cuando SSH no puede entrar**, que son cuatro momentos del curso:

- **Día 1** — instalar el sistema: todavía no existe nada a lo que conectarse
- **Día 5** — si rompen la red: SSH viaja por la red que acaban de romper
- **Día 6** — si el `/etc/fstab` queda mal: el servidor arranca en modo emergencia, sin red
- **Día 10** — recuperar la contraseña de root: pasa antes de que arranque el sistema

La consola es el cable de seguridad. Se usa poco, pero cuando se necesita
no hay sustituto.

---

## Diferencias entre mi pantalla y la de ellos

**El árbol de carpetas es idéntico.** `/etc`, `/var`, `/home`, `/srv` y todo lo
demás está igual en las dos máquinas. Eso conviene decirlo, porque tranquiliza.

Lo que sí cambia son tres cosas:

| Qué | Yo (Mac, UTM) | Ellos (Windows, VirtualBox) |
|---|---|---|
| Discos | `/dev/vda`, `/dev/vdb` | `/dev/sda`, `/dev/sdb` |
| Interfaz de red | `enp0s1` | `enp0s3` |
| Arquitectura (`uname -m`) | `aarch64` | `x86_64` |

**Por qué los discos se llaman distinto:** VirtualBox emula un controlador SATA
y esos discos se llaman `sd`. UTM usa VirtIO y se llaman `vd`. Es el tipo de
controlador, no el sistema.

**Dónde va a aparecer cada una:**

- Los discos, en `lsblk` (Día 1), en `/etc/fstab` y en toda la Jornada 6
- La interfaz de red, en `ip a` (Día 1) y en toda la Jornada 5
- La arquitectura, en `uname -m` (Día 1)

**Qué decir cuando aparezca:** anunciarlo en voz alta la primera vez, explicar
que es por el virtualizador y que los comandos son exactamente los mismos.
Mejor adelantarse que dejar que alguien note la diferencia y se desconcierte.
