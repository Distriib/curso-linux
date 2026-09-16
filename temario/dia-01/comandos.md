# Comandos básicos — Día 1

Todos se ejecutan desde SSH: `ssh -p 2222 student@localhost`

---

## Quién soy y dónde estoy

| Comando | Qué hace |
|---|---|
| `whoami` | Con qué usuario estoy trabajando |
| `id` | Mi número de usuario (UID), mi grupo y a qué otros grupos pertenezco |
| `pwd` | En qué carpeta estoy parado ahora mismo |
| `hostname` | El nombre del servidor |

---

## Qué máquina es esta

| Comando | Qué hace |
|---|---|
| `hostnamectl` | Resumen completo: nombre, sistema operativo, kernel, arquitectura |
| `cat /etc/os-release` | Qué distribución y versión es |
| `cat /etc/redhat-release` | Lo mismo, en una sola línea |
| `uname -r` | Versión del kernel |
| `uname -m` | Arquitectura: `aarch64` en el Mac, `x86_64` en Windows |
| `lscpu` | Procesadores: cuántos núcleos, qué modelo |
| `uptime` | Cuánto lleva encendido y qué tan cargado está |
| `date` | Fecha y hora |
| `timedatectl` | Fecha, hora, zona horaria y si está sincronizado |

---

## Recursos: disco, memoria, red

| Comando | Qué hace |
|---|---|
| `df -h` | Cuánto espacio de disco hay y cuánto queda libre |
| `free -m` | Memoria RAM usada y disponible, en megas |
| `lsblk` | Los discos y sus particiones, en forma de árbol |
| `ip a` | Las tarjetas de red y sus direcciones IP |
| `ping -c 3 redhat.com` | Comprobar que hay salida a internet. `-c 3` = solo 3 intentos |

---

## Moverse por el sistema

| Comando | Qué hace |
|---|---|
| `ls` | Listar lo que hay en la carpeta actual |
| `ls -la` | Listar todo, incluidos los archivos ocultos, con detalles |
| `ls -l /` | Ver la raíz del sistema |
| `cd /etc` | Entrar a una carpeta |
| `cd ..` | Subir un nivel |
| `cd` | Volver a mi carpeta personal |
| `cd -` | Volver a la carpeta anterior |
| `tree -L 1 /` | La raíz en forma de árbol, un solo nivel |

---

## Pedir ayuda

| Comando | Qué hace |
|---|---|
| `man ls` | El manual completo de un comando. Se sale con `q` |
| `man man` | El manual sobre cómo usar los manuales |
| `man 5 passwd` | El manual de la **sección 5** (formatos de archivo), no del comando |
| `man -k hostname` | Buscar en todos los manuales que mencionen esa palabra |
| `ls --help` | Ayuda rápida, más corta que el manual |

Dentro de `man`: flechas para moverse, `/palabra` para buscar, `q` para salir.

---

## Atajos y historial

| Atajo | Qué hace |
|---|---|
| `Tab` | Autocompleta lo que estás escribiendo |
| `↑` `↓` | Recorrer comandos anteriores |
| `Ctrl+C` | Cancelar lo que está corriendo |
| `Ctrl+L` | Limpiar la pantalla |
| `history` | Ver todos los comandos que escribiste |
| `sudo !!` | Repetir el comando anterior, esta vez con permisos de administrador |

---

## Permisos de administrador

| Comando | Qué hace |
|---|---|
| `sudo <comando>` | Ejecutar un solo comando como administrador |
| `sudo -i` | Convertirse en root hasta que se escriba `exit` |
| `exit` | Salir de root, o cerrar la sesión SSH |

Probar la diferencia:

```bash
cat /etc/shadow        # Permission denied
sudo cat /etc/shadow   # funciona
```

Con `sudo -i` el prompt cambia de `$` a `#`. Ese `#` significa que sos root y
que cualquier error se lo lleva todo por delante.

---

## Apagar y reiniciar

| Comando | Qué hace |
|---|---|
| `sudo systemctl reboot` | Reiniciar el servidor |
| `sudo systemctl poweroff` | Apagar el servidor |

Qué pasa con la sesión SSH:

- **Al reiniciar**, la conexión SSH se corta y aparece `Connection closed`.
  La VM se enciende sola en un minuto. Después hay que volver a conectarse
  con `ssh -p 2222 student@localhost`.
- **Al apagar**, la conexión también se corta, pero la VM queda apagada.
  Hay que encenderla desde UTM antes de poder conectarse otra vez.

En ambos casos la ventana de UTM sigue mostrando lo que pasa en la máquina.

---

## Snapshot

No es un comando de Linux: se hace desde UTM, con la **VM apagada**.

1. Apagar: `sudo systemctl poweroff`
2. En UTM, clic derecho sobre `rhel01`
3. Elegir la opción de clonar o guardar el estado
4. Nombrarlo `dia01-fin`

El snapshot se guarda junto al archivo de la máquina virtual, dentro de la
carpeta de UTM en el Mac. Ocupa espacio de disco, así que conviene ir borrando
los viejos a medida que avanza el curso.

Sirve para volver atrás si algo se rompe: se restaura y la máquina queda
exactamente como estaba al final de esa jornada.
