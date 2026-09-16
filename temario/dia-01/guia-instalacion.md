# Guía de instalación de RHEL 9.8 — pantalla por pantalla

Para el instructor. Cada pantalla trae qué elegir y qué explicar si preguntan.

**Vale igual para todos.** El instructor usa UTM en Mac (aarch64) y los
participantes VirtualBox en Windows (x86_64). Las pantallas del instalador son
idénticas; solo cambia el texto de la arquitectura.

---

## Pantalla 1 — Menú de arranque (GRUB)

Fondo negro con cuatro opciones y una cuenta regresiva de 60 segundos.
Si nadie toca nada, arranca sola la opción resaltada.

| Opción | Qué hace |
|---|---|
| **Install Red Hat Enterprise Linux 9.8** | Instala directo. **Esta es la que usamos en clase** |
| **Test this media & install** | Verifica que la ISO no esté corrupta antes de instalar. Añade 5 a 10 min. Viene seleccionada por defecto |
| **Install in FIPS mode** | Activa solo criptografía certificada por el estándar federal de EE.UU. (FIPS 140-3) |
| **Troubleshooting** | Submenú de rescate: arranque con drivers básicos, modo recuperación |

**Qué elegir:** la primera. Flecha arriba y Enter.

**Si preguntan por "Test this media":** verifica la integridad del archivo
descargado. Útil si la descarga se cortó o si el disco viene de un USB dudoso.
En clase no lo usamos por tiempo, pero si una instalación falla de forma rara,
esa es la primera sospecha.

**Si preguntan por FIPS:** es un modo que restringe el sistema a algoritmos
criptográficos certificados para entornos gubernamentales de Estados Unidos.
Suena atractivo para una institución pública, pero tiene costo: deshabilita
algoritmos SSH de uso común y complica la administración. No se usa en el curso.
Es una decisión de política institucional, no técnica, y se toma antes de
instalar porque después no se activa fácilmente.

**Si preguntan por Troubleshooting:** ahí vive el modo rescate, que se usa
cuando un sistema ya instalado no arranca. Se ve en la Jornada 10.

> ⚠️ La cuenta regresiva es fácil de pasar por alto. Avisar al grupo antes de
> llegar a esta pantalla, o van a terminar con la opción equivocada.

---


## Pantalla — Installation Summary

Pantalla central. Nada toca el disco hasta pulsar *Begin Installation*.
Los iconos con ⚠️ naranja son obligatorios.

### SOFTWARE

**Connect to Red Hat** — Registra la suscripción durante la instalación.
Lo dejamos sin registrar y lo hacemos después por terminal con
`subscription-manager`, para que vean el comando.

**Installation Source** — De dónde salen los paquetes. Dice *Local media*
porque vienen del DVD. Si se usara el Boot ISO, aquí habría una URL de internet.

**Software Selection** — Qué se instala. 🔴 **Cambiar a "Server"**.
El default es *Server with GUI*, que instala escritorio GNOME: consume memoria,
amplía la superficie de ataque y no es lo que hay en un servidor real.

### SYSTEM

**Installation Destination** — En qué disco se instala y cómo se particiona.
Automatic crea `/boot`, la raíz y la swap con LVM.

**KDUMP** — Si el kernel se cae, kdump copia el contenido de la memoria RAM en
ese instante a un archivo en disco, para analizar después qué causó la caída.
Es la caja negra del sistema. Se deja **habilitado**, es lo estándar en producción.

**Network & Host Name** — Activa la tarjeta de red y define el nombre del
servidor. ⚠️ En una instalación normal la red viene **apagada**; hay que
encenderla o el sistema queda sin conexión. Aquí también se pone `rhel01`.

**Security Profile** — Aplica automáticamente un perfil de endurecimiento
durante la instalación. Se deja **sin perfil**: el endurecimiento se hace a mano
en la Jornada 8, que es donde se entiende lo que se está aplicando.

Los perfiles que aparecen en la lista, por si preguntan qué son:

| Sigla | Qué es |
|---|---|
| **CIS** | *Center for Internet Security*. Organización sin fines de lucro que publica guías de configuración segura para casi todo: Linux, Windows, bases de datos, nube. Es el estándar de referencia más usado y el más aplicable a una institución pública |
| **PCI-DSS** | *Payment Card Industry Data Security Standard*. Obligatorio para cualquier sistema que procese o almacene datos de tarjetas de crédito. Lo exige la industria de medios de pago, no un gobierno |
| **STIG** | *Security Technical Implementation Guide*. Lo publica la agencia de defensa de Estados Unidos (DISA) para sus propios sistemas. Es el más estricto de los tres |
| **HIPAA** | Estándar de salud de Estados Unidos, para sistemas con historiales médicos |
| **ANSSI** | Guías de la agencia de ciberseguridad de Francia, en varios niveles de exigencia |

Un perfil es, en la práctica, una lista de cientos de reglas concretas: longitud
mínima de contraseña, servicios que deben estar apagados, permisos de archivos,
opciones de montaje. Aplicarlo durante la instalación deja el sistema endurecido
de golpe, pero sin que nadie entienda qué cambió ni por qué.

Por eso en el curso no se aplica ninguno aquí. En la Jornada 8 se usa **OpenSCAP**
para escanear el sistema contra el perfil CIS y leer el informe: así ven qué
reglas existen, cuáles cumplen y cuáles no.

### Qué hay que tocar

| Item | Acción |
|---|---|
| Time & Date | Poner **America/Panama** |
| Software Selection | Cambiar a **Server** |
| Installation Destination | Entrar y dar Done |
| Network & Host Name | Encender la red, hostname `rhel01` |
| Root Password | Definir |
| User Creation | `student`, marcar administrador |

El resto queda por defecto.

---

## Pantalla — Installation Destination

Se selecciona el disco, se deja **Automatic** y se pulsa **Done**. Aunque no se
cambie nada, hay que entrar: si no, queda marcada con ⚠️.

**Encrypt my data** se deja **sin marcar**. Cifra el disco con LUKS y pide una
contraseña en cada arranque; si alguien la olvida, pierde la máquina y el curso.

### El nombre del disco cambia según el hipervisor

| Entorno | Disco |
|---|---|
| **VirtualBox** (participantes, Windows) | `sda` |
| **UTM** (instructor, Mac) | `vda` |

Es la primera vez en el curso que la pantalla del instructor no coincide con la
de los participantes. **Anunciarlo aquí en voz alta**, porque vuelve a aparecer
en `lsblk`, en `/etc/fstab` y en toda la Jornada 6.

La causa es el tipo de controlador de disco virtual: VirtualBox emula SATA
(`sd`), UTM usa VirtIO (`vd`). Los discos adicionales siguen la misma lógica:
`sdb` y `sdc` para ellos, `vdb` y `vdc` para el instructor.

---

## Pantalla — Network & Host Name

La red viene **apagada** en una instalación normal. Encender el interruptor de
la esquina superior derecha; al conectar, toma IP por DHCP y se ven la dirección,
la puerta de enlace y el DNS.

Después, escribir **`rhel01`** en el campo *Host Name* de abajo y pulsar
**Apply**.

> ⚠️ **Sin darle Apply no se aplica.** A la derecha dice *Current host name:
> localhost*; después de Apply tiene que decir `rhel01`. Es un error clásico:
> escriben el nombre, pulsan Done, y el servidor queda llamándose localhost.

Luego **Done**. En esta pantalla no hay nada más que tocar.

### Por qué `rhel01`, y qué es un hostname

La máquina virtual tiene **dos nombres distintos** y conviene no confundirlos.

**El nombre en UTM o VirtualBox** es la etiqueta de la caja. Solo lo ve el
dueño del equipo, en la lista de máquinas de su hipervisor. Como escribir
"Navidad" en una caja de cartón: sirve para encontrarla en el armario, pero la
caja no sabe cómo se llama.

**El hostname** es cómo se llama el servidor a sí mismo, y eso sí lo sabe él.
Funciona igual que el nombre de AirDrop de un iPhone: el teléfono sabe que se
llama "iPhone de Helio" y así se presenta ante los demás.

Dónde aparece el hostname una vez instalado:

- En el prompt de la terminal: `[student@rhel01 ~]$`
- En cada línea de los logs, para saber qué servidor la generó
- En la red, cuando otro equipo lo busca por nombre

Se usa el mismo nombre en los dos lados para no perderse. Con cinco máquinas
virtuales, si la etiqueta y el hostname no coinciden, nadie sabe cuál es cuál.

En producción la convención es institución, función y número: `pgn-web01`,
`pgn-db02`.

### Por qué kdump queda habilitado

Se deja como viene porque es el default de RHEL y lo que hay en producción.
Desactivarlo haría que la máquina virtual deje de parecerse a un servidor real.

Tiene un costo: reserva entre 160 y 250 MB de RAM que quedan apartados. Con
4 GB se nota.

Ese costo es además un buen momento de clase: en la Jornada 4, al correr
`free -m`, la memoria total no dará 4096 sino unos 3800 y pico. La pregunta
"¿por qué falta memoria?" va a salir sola, y la respuesta es kdump. Mejor que
lo vean y lo entiendan a esconderlo desactivándolo.

---

## Pantalla — Root Password

Se define la contraseña de `root`, la cuenta de administración total del sistema.
Ambas casillas quedan **desmarcadas**.

En un laboratorio conviene una contraseña simple y anotada: son máquinas
desechables y perder tiempo de clase por una clave olvidada no aporta nada.
Si es corta, el instalador pide pulsar **Done dos veces**. Recuperar una
contraseña de root perdida es material de la Jornada 10.

### Las dos casillas

**Lock root account** — Deshabilita por completo la cuenta root. El servidor se
administra solo con `sudo`. Algunas organizaciones lo hacen para que nadie pueda
entrar directo como root y todo quede auditado.

> ⚠️ Si se marca y además no se crea un usuario administrador, uno se queda
> afuera de su propio servidor.

**Allow root SSH login with password** — Permite entrar por SSH directamente como
root usando contraseña. Es mala práctica y RHEL 9 lo trae desactivado justamente
por eso: `root` es un nombre de usuario que todo el mundo conoce, así que un
atacante ya tiene la mitad del trabajo hecho y solo le falta adivinar la clave.

---

## Después de instalar — Expulsar la ISO

Al terminar la instalación se pulsa **Reboot System**. Si en vez del sistema
instalado vuelve a aparecer el menú del instalador (*Install Red Hat Enterprise
Linux 9.8*), es que la ISO sigue puesta y la máquina arrancó desde el DVD otra vez.

**Cómo arreglarlo:**

1. Apagar la VM: botón de encendido en la ventana, o cerrarla
2. En la ventana principal de UTM, con `rhel01` seleccionado, mirar abajo donde
   dice **CD/DVD** y muestra `rhel-9.8-aarch64-dvd.iso`
3. Desplegar ese menú y elegir **Clear** (o Eject)
4. Volver a darle play

Es el equivalente a sacar el DVD de la bandeja después de instalar. VirtualBox
suele expulsarla sola; UTM no.

> ⚠️ Si a alguien le pasa en clase, es siempre esto. Se arregla en un minuto,
> pero desconcierta si no se sabe qué mirar.

---

## Primer arranque — Iniciar sesión

Después de expulsar la ISO, el arranque pasa por tres etapas sin que haya que
tocar nada:

1. El menú de GRUB del sistema instalado, unos segundos
2. Texto de arranque pasando rápido
3. La pantalla se queda quieta en `rhel01 login:`

Ahí se escribe `student`, Enter, y después la contraseña.

> Al escribir la contraseña **no se ve nada**: ni asteriscos ni puntos. Es
> normal en Linux y es de las primeras preguntas que salen. La terminal sí está
> recibiendo lo que se escribe.

---

## Registrar la suscripción

```bash
sudo subscription-manager register
```

Son dos partes:

**`sudo`** — Ejecutar como administrador. En Linux los comandos que cambian el
sistema necesitan permisos elevados. Pide la contraseña del usuario.

**`subscription-manager register`** — Conecta ese servidor con la cuenta de
Red Hat.

**Por qué hace falta.** RHEL viene "sin tienda". Recién instalado, al intentar
instalar cualquier programa responde que no hay de dónde bajarlo. Registrar la
máquina la conecta a los repositorios de Red Hat, que es donde vive el software.

Se parece a activar Windows, pero en vez de desbloquear funciones desbloquea el
acceso al software.

Pide el usuario y la contraseña de la cuenta de Red Hat Developer, y responde:

```
The system has been registered with ID: ...
y veremos el msj que solo sale una vez de bienvenida
```

> ⚠️ **Es el bloqueo número uno del primer día.** Alguien intenta instalar algo,
> le sale `This system is not registered with an entitlement server`, y no
> entiende por qué. La respuesta siempre es esta.


---

dfn replo list para comprobar despues
## Comprobar los repositorios

```bash
dnf repolist
```

Tienen que aparecer dos:

| Repositorio | Qué contiene |
|---|---|
| **BaseOS** | El sistema operativo en sí: kernel, systemd, utilidades básicas. Es lo que hace que la máquina funcione. Se mantiene estable durante los 10 años de vida de RHEL 9 |
| **AppStream** | Las aplicaciones que corren encima: servidores web, bases de datos, lenguajes de programación, herramientas. Se actualiza más seguido y puede ofrecer varias versiones de un mismo programa |

La forma corta de explicarlo: **BaseOS es el sistema, AppStream es lo que corre
sobre el sistema.**

Esa separación existe desde RHEL 8. Antes todo venía en un solo repositorio.
Se dividió para que las aplicaciones puedan avanzar de versión sin obligar a
cambiar el sistema operativo por debajo.

> Si `dnf repolist` no muestra nada, la máquina no quedó registrada. Volver al
> paso anterior.

---

## Actualizar el sistema

```bash
sudo dnf -y update
```

Es lo primero que se hace en cualquier servidor recién instalado. La ISO se
generó hace meses: entre esa fecha y hoy salieron parches de seguridad y
correcciones que no están en el disco.

Qué hace el comando: consulta los repositorios, compara con lo instalado,
descarga lo que cambió y lo aplica. El `-y` responde "sí" a la confirmación,
para no tener que estar atento.

Puede tardar entre 10 y 30 minutos según la conexión, y descarga varios cientos
de megas. Corre solo, así que conviene lanzarlo y seguir con otra cosa mientras.

> ⚠️ **En clase, lanzarlo temprano.** Con cuatro personas descargando a la vez
> por la misma red institucional, el tiempo se multiplica. El mejor momento es
> dejarlo corriendo antes del descanso.

Si el update instaló un kernel nuevo, hace falta reiniciar para usarlo. El
sistema sigue funcionando mientras tanto, pero con el kernel viejo.

---

## Si el update instaló un kernel nuevo

Al terminar `dnf update` suele aparecer, entre los paquetes instalados:

```
Installed:
  kernel-5.14.0-687.46.1.el9_8.aarch64
  kernel-core-...  kernel-modules-...
```

El sistema **sigue corriendo con el kernel viejo**. El nuevo está en el disco
esperando el próximo arranque.

**Cómo demostrarlo en 2 minutos:**

```bash
uname -r                    # muestra el kernel viejo
sudo systemctl reboot       # reiniciar
# volver a conectarse y repetir
uname -r                    # ahora muestra el nuevo
```

### Cómo explicar qué es un kernel

> El kernel es el corazón del sistema. Es lo que habla con el hardware. Cuando
> ustedes escriben un comando, ese comando no toca el disco directamente: le
> pide al kernel que lo haga.
>
> Red Hat publica kernels nuevos con parches de seguridad y correcciones.
> Actualizar el kernel es como cambiarle el motor al servidor. Y como no se le
> puede cambiar el motor a un carro andando, hay que reiniciar.

### Kernel y versión de RHEL no son lo mismo

Es la confusión más común. **RHEL 9.8 es el modelo del carro; el kernel es el
motor.** Se le puede cambiar el motor por uno más nuevo y sigue siendo el mismo
modelo. Por eso el kernel pasa de `687.5.3` a `687.46.1` y el sistema sigue
siendo RHEL 9.8.

### Los kernels viejos no se borran

RHEL conserva los últimos tres. Aparecen en el menú de GRUB al arrancar.

El arranque normal elige solo el más nuevo, sin que nadie toque nada. Los viejos
solo se usan si el nuevo da problemas: se reinicia, se interrumpe el menú con
una flecha y se elige el anterior.

Es una red de seguridad, no un paso del arranque. Se ve a fondo en la Jornada 10.

---

## Cerrar la jornada — Snapshot

Al final de cada jornada se guarda el estado de la máquina, para poder volver
atrás si algo se rompe después.

### Los participantes (VirtualBox): snapshot

1. Menú **Máquina → Tomar instantánea**
2. Nombre: `dia01-fin`
3. Aceptar

Funciona con la VM encendida o apagada. Queda en una lista de la que después se
restaura con un clic. Guarda solo las diferencias, así que ocupa poco.

### El instructor (UTM): clonar

UTM no tiene snapshots. La alternativa es clonar la máquina completa:

1. Apagar: `sudo systemctl poweroff`
2. En UTM, clic derecho sobre `rhel01` → **Clone**
3. Renombrar el clon a `rhel01-dia01`

Duplica el disco entero, así que no conviene hacer uno por jornada. Basta con
clonar antes de las jornadas que rompen cosas (la 6 y la 10) y borrar el clon
anterior cada vez.

### Qué es un snapshot

Una foto del estado completo de la máquina virtual en un momento dado: disco,
memoria y configuración. Restaurarlo devuelve la máquina exactamente a como
estaba, deshaciendo todo lo que pasó después.

No es un comando de Linux ni algo que el servidor conozca. Lo hace el hipervisor
desde afuera: la máquina virtual ni se entera.

> ⚠️ Insistir en que lo tomen. El que no lo hace lo lamenta en la Jornada 6 o en
> la 10, que son las que rompen el sistema a propósito.
