# Día 01 — Instalar RHEL 9 y primer contacto con la shell

> Al terminar el día, el participante tiene una VM con RHEL 9 "Server" (sin GUI) instalada, registrada con su suscripción Developer, actualizada, accesible por SSH desde su propio equipo y respaldada con el snapshot `dia01-fin`; además sabe leer el prompt, consultar la ayuda (`man`, `--help`) y ejecutar los comandos básicos que identifican un sistema Linux.

**Ficha técnica cubierta:**
- **RH124 M1: Introducción a Linux** — Conceptos básicos, Arquitectura Linux, Shell y terminal, Navegación del sistema (inicio: `pwd`, `cd`, `ls -la`, árbol resumido de `/`; se profundiza el Día 2).
- **RH124 M6: Gestión de Software** — DNF/YUM y Repositorios, solo en su parte introductoria: registro con `subscription-manager`, `dnf repolist`, `dnf install`, `dnf update`. RPM, grupos y repositorios propios se ven completos el Día 6.
- **RH124 M5: Redes Básicas** — SSH, solo el primer acceso como cliente (`ssh -p 2222 student@localhost`) y la lectura de la dirección con `ip a`. La configuración de IP, DNS y el servidor `sshd` se ven el Día 5.
- **RH124 M4: Administración del Sistema** — Servicios, solo lo mínimo para arrancar y apagar el sistema (`systemctl reboot`, `systemctl poweroff`, `systemctl get-default`). Se profundiza el Día 4.

**Requisitos previos:**
- Equipo del participante con al menos 8 GB de RAM y **50 GB de disco libres** (ISO ~10 GB + disco de la VM que crece hasta 20 GB + los snapshots de los diez días).
- Cuenta gratuita en https://developers.redhat.com (Red Hat Developer Subscription for Individuals), creada y con correo confirmado.
- ISO de RHEL 9 descargada **antes de la clase**: `rhel-9.x-x86_64-dvd.iso` (DVD, ~10–11 GB, recomendada) o `rhel-9.x-x86_64-boot.iso` (~1 GB, requiere red y registro durante la instalación). En Mac con chip Apple: `rhel-9.x-aarch64-dvd.iso`.
- Hipervisor instalado: VirtualBox 7.x (7.1 o posterior; Windows y Mac Intel) o UTM 4.x (Mac Apple Silicon).
- Virtualización habilitada en el firmware del equipo (VT-x / AMD-V). Se verifica en vivo en el Lab 2.1.
- Cliente SSH: Windows 10/11 ya incluye `ssh` en PowerShell; macOS lo trae en Terminal.
- No hay snapshot previo: este es el primer día.

---

## Agenda (240 min)

| Minuto | Duración | Bloque | Contenido |
|---|---|---|---|
| 0:00–0:10 | 10 min | Apertura | Presentación del curso y del grupo, reglas de trabajo (checkpoints en el chat, cámara, preguntas), sondeo rápido: quién tiene ISO, hipervisor y cuenta listos |
| 0:10–0:35 | 25 min | Bloque 1 — Conceptos | Qué es Linux, kernel vs distribución vs GNU, familia Red Hat, ciclo de vida de RHEL 9, suscripción Developer, arquitectura de un sistema Linux, VM / hipervisor / ISO, por qué RHEL 9 y no 10 (lo que no alcance se completa mientras Anaconda copia paquetes) |
| 0:35–0:45 | 10 min | Bloque 2 — Conceptos + Lab 2.1 | Conceptos (3 min) y Lab 2.1 (7 min): verificar la preparación: cuenta, ISO y su checksum, virtualización habilitada, conflicto con Hyper-V |
| 0:45–1:05 | 20 min | Bloque 2 — Lab 2.2 | Crear la VM `rhel01` en VirtualBox 7 / UTM: 2 vCPU, 4 GB, 20 GB, red NAT, port forwarding 2222→22 y 8080→80 |
| 1:05–1:50 | 45 min | Bloque 3 — Conceptos + Lab 3.1 | Conceptos (5 min) y Lab 3.1 (40 min): instalación con Anaconda: idioma, teclado, zona horaria, destino automático, red y hostname, "Server" sin GUI, root, `student`; Begin Installation |
| 1:50–2:05 | 15 min | Descanso | La instalación copia paquetes mientras el grupo descansa |
| 2:05–2:35 | 30 min | Bloque 3 — Lab 3.2 | Primer arranque, retirar la ISO, login en consola, `subscription-manager register`, `dnf repolist`, lanzar `dnf update` |
| 2:35–2:50 | 15 min | Bloque 4 — Conceptos + Lab 4.1 | Conceptos (5 min) y Lab 4.1 (10 min): SSH desde el equipo propio: qué es el port forwarding, aceptar la huella, primera sesión remota |
| 2:50–3:35 | 45 min | Bloque 5 — Conceptos + Labs 5.1 y 5.2 | Conceptos (10 min), Lab 5.1 (15 min) y Lab 5.2 (20 min): primer contacto con la shell: prompt `$` vs `#`, identidad, estado del sistema, red, recursos, `man`, `--help`, Tab, historial, atajos, `sudo -i`, navegación inicial |
| 3:35–3:50 | 15 min | Reto + Bloque 6 — Conceptos + Lab 6.1 | Reto: checklist de salida del Día 1 con la VM encendida (5 min); conceptos de snapshot (2 min) y Lab 6.1 (8 min): reinicio con el kernel nuevo, apagado y snapshot `dia01-fin` |
| 3:50–4:00 | 10 min | Cierre | Resumen, cheatsheet, tarea, dudas |

Total: 240 min (10 + 25 + 10 + 20 + 45 + 15 + 30 + 15 + 45 + 15 + 10).

---

## Prioridad si falta tiempo

**Imprescindible**
- VM creada con los parámetros estándar (2 vCPU, 4 GB, 20 GB, NAT con port forwarding 2222→22 y 8080→80).
- RHEL 9 instalado como "Server" SIN GUI, hostname `rhel01`, usuario `student` marcado como administrador, contraseña de root anotada.
- Red activa en la VM y sistema registrado: `sudo subscription-manager register` y `sudo dnf repolist` mostrando BaseOS y AppStream.
- Acceso por SSH desde el equipo propio: `ssh -p 2222 student@localhost`.
- Snapshot `dia01-fin` tomado.
- Comandos mínimos ejecutados por cada participante: `whoami`, `id`, `hostname`, `cat /etc/os-release`, `ip a`, `man ls`, `exit`.

**Importante**
- Conceptos del Bloque 1 completos (familia Red Hat, ciclo de vida, qué es una suscripción, arquitectura por capas).
- `sudo dnf -y update` terminado y reinicio con el kernel nuevo antes del snapshot.
- Labs 5.1 y 5.2 completos: `hostnamectl`, `uname -r`, `df -h`, `free -m`, `lscpu`, secciones de `man`, Ctrl+R, `sudo -i` / `exit`, `ls -la`, árbol de `/`.
- Verificación del checksum de la ISO.
- Qué significa la huella (fingerprint) de SSH y dónde queda guardada.

**Si sobra tiempo**
- Instalador en modo texto (`inst.text`): demo del instructor.
- `rhc connect` como alternativa de registro: demo del instructor.
- Corregir teclado y zona horaria después de instalar: `localectl`, `timedatectl`.
- `sudo dnf -y install bash-completion vim-enhanced` (solo cuando `dnf update` haya terminado: dnf no admite dos transacciones a la vez).
- Cambiar la tecla Host de VirtualBox; usar `Host+F2` para una segunda consola (tty2).
- Si alguien no logró instalar a las 2:35, el instructor entrega la OVA / VM exportada (plan B) para que pueda seguir los Bloques 4 y 5; termina su instalación como tarea.

---

## Bloque 1 — Conceptos: Linux, Red Hat y la máquina virtual

### Conceptos (25 min)

Este bloque no tiene lab: es la única parte "de pizarra" del curso y conviene darla mientras los participantes todavía tienen la atención fresca. Si alguien llega con la ISO sin descargar, que inicie la descarga ahora mismo, mientras escucha. Si a los 25 minutos no se llegó al final, se corta y se retoma lo que falte (ciclo de vida, RHEL 10) mientras Anaconda copia paquetes en el Lab 3.1 o mientras corre `dnf update`.

**Qué es Linux.** En sentido estricto, Linux es solo el **kernel** (núcleo): el programa que arranca primero, habla con el hardware y reparte CPU, memoria, disco y red entre los demás programas. Lo empezó Linus Torvalds en 1991. Un kernel solo no le sirve a nadie: hace falta una shell, comandos (`ls`, `cp`, `grep`), compilador, bibliotecas. La mayoría de esas herramientas vienen del proyecto **GNU** (1983, Richard Stallman), por eso el nombre correcto del conjunto es "GNU/Linux". Cuando se dice "Linux" en una sala de servidores, casi siempre se está hablando de ese conjunto.

**Qué es una distribución.** Una **distribución** (distro) es kernel + herramientas GNU + gestor de paquetes + instalador + política de actualizaciones + (a veces) soporte comercial. Analogía útil: el kernel es el motor; la distribución es el auto completo, con garantía y taller. Ubuntu, Debian, SUSE y Red Hat Enterprise Linux son distribuciones; todas usan el mismo kernel con distinta "carrocería" y distinto gestor de paquetes (`apt` en Debian/Ubuntu, `dnf` en la familia Red Hat, `zypper` en SUSE).

**La familia Red Hat.** Conviene dibujarla como una tubería:

```
Fedora  ──>  CentOS Stream  ──>  RHEL  ──>  Rocky Linux / AlmaLinux / Oracle Linux
(comunidad,   (vista previa      (producto    (rebuilds: reconstrucciones compatibles,
 6 meses)      continua de la     empresarial,   gratuitas, sin soporte de Red Hat)
               próxima RHEL 9.x)  10 años)
```

- **Fedora**: laboratorio. Versión nueva cada seis meses; lo que funciona ahí llega a RHEL dos o tres años después.
- **CentOS Stream**: lo que será la próxima versión menor de RHEL. Ya no es el "CentOS gratis igual a RHEL" que muchos conocieron: ese producto (CentOS Linux) terminó; CentOS Linux 7 quedó sin soporte el 30 de junio de 2024. Si en la institución quedan servidores CentOS 7, están sin parches de seguridad.
- **RHEL**: el producto. Estable, certificado, con soporte de diez años.
- **Rocky Linux y AlmaLinux**: nacieron en 2021 para reemplazar a CentOS Linux; son reconstrucciones binariamente compatibles. Todo lo que se aprende en este curso funciona igual en ellas, salvo el registro con `subscription-manager`.

**Ciclo de vida de RHEL 9 y qué significa "soporte".** RHEL 9 salió en mayo de 2022 (nombre en clave "Plow"). Cada seis meses aparece una versión menor: 9.0, 9.1 … 9.4 (2024), 9.6 (2025), etc. Las fases:

| Fase | Hasta | Qué incluye |
|---|---|---|
| Full Support | mayo 2027 | Parches de seguridad y errores, soporte de hardware nuevo, funcionalidades nuevas |
| Maintenance Support | 31 de mayo de 2032 | Parches de seguridad y correcciones críticas; sin funcionalidades nuevas |
| Extended Life Cycle Support (ELS) | ~2035 (complemento de pago) | Solo parches críticos |

"Soporte" significa tres cosas: (1) acceso a los repositorios con parches firmados y trazables (cada corrección de seguridad es una errata RHSA con su CVE), (2) derecho a abrir casos con ingenieros de Red Hat, y (3) certificación de hardware y software de terceros (bases de datos, respaldo, SAP). Para una entidad pública, los puntos 1 y 3 son los que justifican una suscripción: auditoría y cumplimiento.

**Qué es una suscripción.** RHEL no se "compra con licencia": se **suscribe**. La suscripción da acceso al contenido (repositorios) y al soporte durante un año renovable. Sin suscripción el sistema arranca y funciona, pero `dnf` no tiene de dónde bajar paquetes. Este curso usa la **Red Hat Developer Subscription for Individuals**: gratuita, hasta 16 sistemas, para desarrollo y pruebas (no para producción), se renueva cada año volviendo a aceptar los términos en developers.redhat.com.

Detalle importante para no perderse con tutoriales viejos: las cuentas actuales usan **Simple Content Access (SCA)**. Antes había que registrar el sistema y además "adjuntar" (attach) una suscripción. Hoy basta `subscription-manager register`; el acceso a contenido lo da la cuenta, no una suscripción adjunta. Si un tutorial dice `subscription-manager attach --auto`, ese paso ya no aplica.

**Arquitectura de un sistema Linux.** Capas, de abajo hacia arriba:

```
 ┌──────────────────────────────────────────────────────┐
 │  Aplicaciones y shell: bash, vim, dnf, ssh, httpd…    │  ← aquí trabaja el participante
 ├──────────────────────────────────────────────────────┤
 │  systemd (PID 1) y servicios: sshd, NetworkManager,   │
 │  chronyd, firewalld, rsyslog…                         │
 ├──────────────────────────────────────────────────────┤
 │  Kernel Linux: procesos, memoria, sistema de archivos,│
 │  red, controladores (drivers), SELinux                 │
 ├──────────────────────────────────────────────────────┤
 │  Hardware real o hardware virtual que ofrece el       │
 │  hipervisor: CPU, RAM, disco, tarjeta de red          │
 └──────────────────────────────────────────────────────┘
```

Qué decir en clase: el usuario nunca habla con el kernel directamente. Habla con la **shell** (bash), que interpreta lo que escribe y lanza programas; los programas piden servicios al kernel (abrir un archivo, enviar un paquete) mediante llamadas al sistema. **systemd** es el primer proceso que el kernel arranca (PID 1) y es quien inicia todo lo demás: red, SSH, hora, firewall. Cuando el Día 4 se vea `systemctl`, es esta capa.

**VM, hipervisor tipo 2, ISO, snapshot.**
- Una **máquina virtual** es una computadora simulada por software: tiene su CPU, RAM, disco y tarjeta de red virtuales, y dentro corre un sistema operativo completo que cree estar en hardware real.
- El **hipervisor** es el programa que la ejecuta. Tipo 1 (bare metal): VMware ESXi, Microsoft Hyper-V, KVM (el hipervisor de Linux, usado por Red Hat en OpenShift Virtualization). Tipo 2 (sobre un sistema de escritorio): VirtualBox, UTM, VMware Workstation/Fusion. En el curso usamos tipo 2; los conceptos son idénticos.
- Una **ISO** es la imagen de un DVD de instalación en un solo archivo. En la VM se "inserta" en la unidad óptica virtual y la VM arranca desde ella.
- Un **snapshot** es una foto del estado completo de la VM. Permite romper el sistema sin miedo y volver atrás en segundos. Se tomará uno al final de cada día.

**RHEL 10 y por qué el curso usa RHEL 9.** RHEL 10 salió en mayo de 2025 (kernel 6.12, exige procesadores x86_64-v3, elimina Xorg, entre otros cambios). El curso usa RHEL 9 porque: (1) es lo que hay en producción en la mayoría de instituciones y lo que van a administrar; (2) su documentación, foros y soluciones de la base de conocimiento están maduros; (3) el examen RHCSA (EX200) se ofrece sobre RHEL 9 y los objetivos son prácticamente los mismos en RHEL 10; (4) todo lo que se aprende aquí (dnf, systemd, NetworkManager, firewalld, SELinux, LVM, Podman) es igual en RHEL 10. Cambios que conviene tener presentes porque aparecen en foros: `yum` es solo un enlace a `dnf` (ya lo era en RHEL 8, pero mucha documentación sigue escribiendo `yum`); se eliminó el paquete `network-scripts` con sus `ifup`/`ifdown`, y aunque NetworkManager todavía lee los `ifcfg-*` heredados, el formato propio de RHEL 9 son los **keyfiles** `/etc/NetworkManager/system-connections/*.nmconnection` (Día 5); cgroups v2 por defecto (en RHEL 8 era v1); OpenSSL 3, que rechaza firmas y certificados SHA-1; root ya no entra por SSH con contraseña; Podman 4/5 sin Docker.

---

## Bloque 2 — Preparación y creación de la VM

### Conceptos (3 min)

Antes de instalar hay que confirmar tres cosas: que la ISO bajó completa (checksum), que el procesador tiene la virtualización habilitada (sin VT-x/AMD-V ningún hipervisor arranca un sistema de 64 bits) y que el hipervisor está instalado en la versión correcta. En Windows hay un matiz: Hyper-V (y con él WSL2 y Docker Desktop) toma el control de la virtualización; VirtualBox 7 puede convivir con Hyper-V usando la "Windows Hypervisor Platform", pero más lento. Se explica cómo detectarlo y cómo convivir.

### Lab 2.1 — Verificar la preparación previa (7 min)

- **Objetivo:** confirmar que cada participante tiene cuenta activa, ISO íntegra e hipervisor capaz de virtualizar antes de crear la VM.

1. **Cuenta Red Hat.** Abrir https://developers.redhat.com e iniciar sesión. Luego abrir https://access.redhat.com/management/subscriptions.

   **Salida esperada:** la página de suscripciones lista "Red Hat Developer Subscription for Individuals".

   Qué observar: si no aparece, en developers.redhat.com/products/rhel/download pulsar el botón de descarga; al hacerlo la primera vez se aceptan los términos y se activa la suscripción. Anotar el **nombre de usuario** (Red Hat login), no solo el correo: `subscription-manager` pide ese nombre.

2. **Integridad de la ISO.** Comparar el SHA-256 con el publicado en la página de descarga (enlace "Checksum"). Sustituir `9.x` por la versión descargada.

   Windows (PowerShell):
   ```powershell
   Get-FileHash -Algorithm SHA256 "$HOME\Downloads\rhel-9.x-x86_64-dvd.iso"
   ```
   macOS (Terminal):
   ```bash
   shasum -a 256 ~/Downloads/rhel-9.x-aarch64-dvd.iso
   ```
   **Salida esperada:** una cadena hexadecimal de 64 caracteres idéntica a la del portal. Tarda 1–3 minutos con la ISO de 10 GB.

   Qué observar: si no coincide, la descarga se cortó; hay que volver a descargar (o copiar del USB/enlace del instructor).

3. **Virtualización habilitada.**

   Windows: abrir el Administrador de tareas (Ctrl+Shift+Esc) → pestaña Rendimiento → CPU → leer la línea **"Virtualización: Habilitado"**. Si dice "Deshabilitado": reiniciar, entrar al BIOS/UEFI (F2, F10, Supr según el fabricante) y activar "Intel Virtualization Technology" / "VT-x" o "SVM Mode" / "AMD-V".

   Windows, comprobar si Hyper-V está tomando la virtualización (PowerShell como administrador):
   ```powershell
   bcdedit | findstr /i hypervisorlaunchtype
   ```
   **Salida esperada:** `hypervisorlaunchtype    Auto` (Hyper-V activo: VirtualBox funcionará en modo compatible, con un ícono de tortuga en su barra de estado), `hypervisorlaunchtype    Off` (VirtualBox usa VT-x directamente, más rápido) o **ninguna salida**, que es lo más común y significa que Hyper-V no está instalado: también es el caso rápido.

   Qué observar: si VirtualBox falla con `VERR_NEM_VM_CREATE_FAILED` o la VM es inusablemente lenta, desactivar Hyper-V con `bcdedit /set hypervisorlaunchtype off` y reiniciar (WSL2 y Docker Desktop dejan de funcionar mientras esté en `off`; se revierte con `bcdedit /set hypervisorlaunchtype auto`).

   macOS (Intel o Apple Silicon):
   ```bash
   sysctl kern.hv_support
   uname -m
   ```
   **Salida esperada:** `kern.hv_support: 1` y `x86_64` (Mac Intel → VirtualBox e ISO x86_64) o `arm64` (Apple Silicon → UTM e ISO aarch64).

4. **Versión del hipervisor.** VirtualBox: menú Ayuda → Acerca de: 7.1.x (mínimo 7.0). UTM: menú UTM → About UTM: 4.x.

- **Checkpoint:** pegar en el chat una línea con: sistema del equipo, hipervisor y versión, arquitectura de la ISO y los primeros 8 caracteres del SHA-256. Ejemplo: `Windows 11 / VirtualBox 7.1.4 / x86_64 dvd / 3f9a1c0e`.

### Lab 2.2 — Crear la VM `rhel01` (20 min)

- **Objetivo:** tener la VM creada con la ISO conectada, 2 vCPU, 4 GB de RAM, disco de 20 GB, red NAT y port forwarding 2222→22 y 8080→80.

**VirtualBox 7 (Windows y Mac Intel)**

1. Máquina → Nueva. Campos:
   - **Nombre:** `rhel01`. Carpeta: la predeterminada.
   - **Imagen ISO:** seleccionar `rhel-9.x-x86_64-dvd.iso`. VirtualBox detecta "Red Hat (64-bit)".
   - **Marcar "Omitir instalación desatendida" (Skip Unattended Installation).** Si no se marca, VirtualBox intenta instalar solo con un usuario `vboxuser` y arruina la práctica.
2. Sección **Hardware:** Memoria base `4096` MB (mínimo 3072). Procesadores `2`. "Habilitar EFI": opcional; se recomienda dejarlo **sin marcar** para que la tabla de particiones coincida con la que se explica en clase.
3. Sección **Disco duro:** "Crear un disco duro virtual ahora", tamaño `20` GB, tipo VDI, **sin** marcar "Reservar tamaño completo" (dinámico). Terminar.

   **Resultado esperado:** la VM aparece en la lista, apagada, con la ISO montada en el controlador IDE.

4. Con la VM seleccionada: Configuración → **Red** → Adaptador 1: "Habilitar adaptador de red", Conectado a: **NAT**. Desplegar **Avanzadas** → botón **Reenvío de puertos** → agregar dos reglas con el ícono "+":

   | Nombre | Protocolo | IP anfitrión | Puerto anfitrión | IP invitado | Puerto invitado |
   |---|---|---|---|---|---|
   | ssh | TCP | 127.0.0.1 | 2222 | (vacío) | 22 |
   | http | TCP | 127.0.0.1 | 8080 | (vacío) | 80 |

   Qué observar: "IP anfitrión" en `127.0.0.1` hace que solo el propio equipo pueda conectarse a la VM; si se deja vacío, cualquier equipo de la red del participante podría llegar al puerto 2222. Buena práctica de seguridad desde el primer día.

5. Configuración → **Sistema** → pestaña Placa base: orden de arranque con "Óptica" antes que "Disco duro" (así viene por defecto). Configuración → **Almacenamiento**: confirmar que bajo "Controlador: IDE" está la ISO; si dice "Vacío", clic en el ícono del disco a la derecha → "Seleccionar un archivo de disco…".
6. Opcional: Configuración → Audio → desmarcar "Habilitar audio" (evita advertencias en el log de la VM).

**UTM (Mac Apple Silicon — equipo del instructor)**

1. Abrir UTM → "Create a New Virtual Machine" → **Virtualize** (no "Emulate": emular x86_64 sería diez veces más lento).
2. Operating System: **Linux**.
3. Pantalla Linux: dejar **sin marcar** "Use Apple Virtualization" (con el backend QEMU se dispone de "Emulated VLAN" con port forwarding y de discos qcow2 para snapshots). "Boot ISO Image" → Browse → `rhel-9.x-aarch64-dvd.iso`. Continue.
4. Hardware: Memory `4096` MB, CPU Cores `2`. Continue.
5. Storage: `20` GB. Continue.
6. Shared Directory: dejar vacío. Continue.
7. Summary: Name `rhel01`, marcar "Open VM Settings". Save.
8. En Settings → **Network**: Network Mode **Emulated VLAN** (en lugar de "Shared Network"). Con este modo la VM recibe `10.0.2.15`, igual que en VirtualBox, y aparece la pestaña **Port Forward** (⚠️ Verificar en la VM antes de la clase: en algunas versiones de UTM la pestaña solo aparece después de guardar los ajustes y volver a abrirlos). Agregar:
   - New → Protocol TCP, Guest Address (vacío), Guest Port `22`, Host Address `127.0.0.1`, Host Port `2222`.
   - New → Protocol TCP, Guest Address (vacío), Guest Port `80`, Host Address `127.0.0.1`, Host Port `8080`.

   Qué observar: la alternativa es dejar "Shared Network": la VM recibe una IP `192.168.64.x` alcanzable directamente desde el Mac (`ssh student@192.168.64.x`) sin port forwarding, pero la IP puede cambiar y los comandos dejan de coincidir con los del grupo. Para la clase se prefiere Emulated VLAN.
9. Settings → **Drives**: debe haber un "VirtIO Drive" de 20 GB (será `/dev/vda`) y una unidad removible con la ISO. Save.

**Ambos hipervisores: comprobación final**

- Encender la VM (VirtualBox: Iniciar; UTM: ▶). Debe aparecer el menú de arranque de RHEL (ver Lab 3.1, paso 1). Apagarla o dejarla en ese menú hasta que todo el grupo llegue.
- Tecla para liberar el teclado y el mouse de la ventana de la VM: VirtualBox **Ctrl derecho** en Windows y **⌘ izquierda** en Mac Intel (tecla Host, visible abajo a la derecha de la ventana); UTM **Control+Option** (⌃⌥).

- **Checkpoint:** pegar en el chat la regla de port forwarding tal como se escribió, por ejemplo `ssh TCP 127.0.0.1:2222 -> 22 ; http TCP 127.0.0.1:8080 -> 80`, y la frase "menú de RHEL visible".

---

## Bloque 3 — Instalación de RHEL 9 con Anaconda

### Conceptos (5 min)

**Anaconda** es el instalador de Fedora, CentOS Stream y RHEL. Su pantalla principal, "Installation Summary", funciona como un tablero (hub-and-spoke): se entra a cada sección, se configura, se pulsa "Done" y se vuelve al tablero. Las secciones con un triángulo naranja son obligatorias; el botón "Begin Installation" se habilita cuando no queda ninguna con advertencia. El orden en que se completan no importa, pero conviene activar la red antes que la hora (para NTP) y antes que "Connect to Red Hat".

Tres decisiones del instalador que hay que tomar bien porque cuestan caro corregirlas después:

- **Software Selection:** el valor por defecto es "Server with GUI". El curso exige "Server" (sin escritorio). Un servidor institucional no tiene escritorio gráfico: consume RAM, agranda la superficie de ataque y no se administra con el mouse.
- **Installation Destination:** particionado **automático**. Anaconda crea `/boot` (1 GiB, XFS, partición estándar) y un grupo de volúmenes LVM llamado `rhel` con dos volúmenes lógicos: `root` (XFS, casi todo el disco) y `swap` (según la RAM). LVM es una capa entre las particiones y los sistemas de archivos que permite crecer y mover volúmenes en caliente; se estudia el Día 6. Hoy basta con reconocer esos nombres en `lsblk`.
- **User Creation:** el usuario `student` debe quedar marcado como administrador (pertenece al grupo `wheel` y puede usar `sudo`). Si se olvida, después hay que entrar como root para arreglarlo.

Idioma: se recomienda instalar en **inglés** aunque el curso sea en español. Los mensajes de error en inglés son los que aparecen en la documentación, en los foros y en la base de conocimiento de Red Hat; buscar un error traducido casi nunca da resultados. El teclado sí debe ser el físico del participante (Latinoamericano, Español o US), si no la contraseña se escribe distinta de como se cree.

### Lab 3.1 — Instalar RHEL 9 con Anaconda (40 min)

- **Objetivo:** dejar RHEL 9 "Server" instalado con hostname `rhel01`, red activa, zona horaria America/Panama, root con contraseña conocida y `student` administrador.

1. **Arrancar la VM** desde la ISO. Aparece el menú GRUB:
   ```text
   Install Red Hat Enterprise Linux 9.x
   Test this media & install Red Hat Enterprise Linux 9.x      <- resaltada por defecto (60 s)
   Troubleshooting -->
   ```
   Con la flecha arriba elegir **"Install Red Hat Enterprise Linux 9.x"** y pulsar Enter (la verificación del medio tarda varios minutos y ya se comprobó el checksum).

   **Salida esperada:** mensajes del kernel en texto y, tras 30–90 segundos, la pantalla gráfica "WELCOME TO RED HAT ENTERPRISE LINUX 9.x".

   Qué observar: si la pantalla queda negra más de dos minutos (ocurre a veces en UTM), reiniciar la VM, en el menú GRUB pulsar `e`, ir al final de la línea que empieza con `linux` (o `linuxefi`), agregar un espacio y `inst.text`, y pulsar Ctrl+X. Arranca el instalador en modo texto, con un menú numerado: `1) Language settings … 9) User creation`; se elige cada número, se configura, y al final se pulsa `b` para comenzar. Las opciones son las mismas que en modo gráfico.

2. **Idioma:** en la lista izquierda "English", en la derecha "English (United States)". Continue.

   **Salida esperada:** pantalla "INSTALLATION SUMMARY" con cuatro grupos: LOCALIZATION (Keyboard, Language Support, Time & Date), SOFTWARE (Connect to Red Hat, Installation Source, Software Selection), SYSTEM (Installation Destination, KDUMP, Network & Host Name, Security Profile) y USER SETTINGS (Root Password, User Creation).

3. **Keyboard.** Clic en "Keyboard" → botón "+" → escribir `Spanish` → seleccionar **"Spanish (Latin American)"** → Add. Seleccionarla y pulsar la flecha "^" para dejarla de primera. Dejar también "English (US)" en la lista, en segundo lugar. Probar en el campo "Test the layout configuration below:" escribiendo `@ - _ / ñ`. Done.

   Qué observar: en el teclado latinoamericano `@` se escribe con Alt Gr + Q; en el español de España con Alt Gr + 2. Quien tenga teclado de España elige "Spanish (Spain)"; quien tenga teclado en inglés deja solo "English (US)". La contraseña que se escriba más adelante usa esta distribución.

4. **Time & Date.** Region: `Americas`. City: `Panama`. Interruptor "Network Time" en ON (si está gris, primero activar la red en el paso 6 y volver). Done.

   **Salida esperada:** el tablero muestra "Americas/Panama timezone".

5. **Installation Destination.** Aparece el disco: en VirtualBox `ATA VBOX HARDDISK  20 GiB  sda`; en UTM un disco VirtIO de `20 GiB  vda` (el texto descriptivo exacto varía; ⚠️ Verificar en la VM antes de la clase). Clic sobre el disco para que muestre la marca de selección. Storage Configuration: **Automatic**. Done.

   **Salida esperada:** regresa al tablero sin diálogos (el disco está vacío) y la sección dice "Automatic partitioning selected".

   Qué observar: con "Custom" se vería el esquema que Anaconda propone: `/boot` 1 GiB, `rhel-root` y `rhel-swap`. No hace falta entrar; se verá con `lsblk` después de instalar.

6. **Network & Host Name.** En la lista aparece "Ethernet (enp0s3)" (en UTM el nombre será distinto, por ejemplo `enp0s1`; verificar después con `nmcli device`). Poner el interruptor de la derecha en **ON**.

   **Salida esperada:** debajo de la interfaz: `Connected`, "Hardware Address 08:00:27:…", "Speed 1000 Mb/s", "IP Address 10.0.2.15/24", "Default Route 10.0.2.2", "DNS 10.0.2.3".

   En el campo "Host Name" (abajo a la izquierda) escribir `rhel01` y pulsar **Apply**. Debe leerse "Current host name: rhel01". Done.

   Qué observar: `10.0.2.15` es la dirección que asigna la red NAT tanto en VirtualBox como en UTM (modo Emulated VLAN). `10.0.2.2` es el "router" virtual y `10.0.2.3` el DNS que el hipervisor ofrece. Se estudia el Día 5.

7. **KDUMP.** Desmarcar "Enable kdump". Done. Es opcional: kdump reserva ~200 MB de RAM para volcados de memoria del kernel, útil en producción, innecesario en la VM del curso.

8. **Connect to Red Hat.** Hoy se **omite** (se registra después de instalar, en el Lab 3.2, cuando se puede ver mejor qué ocurre). Si un participante usa la **boot ISO** (1 GB), este paso es obligatorio para él porque no tiene paquetes locales: Authentication "Account", Username y Password de Red Hat, desmarcar "Connect to Red Hat Insights" si se desea, y pulsar **Register**. Al registrarse, "Installation Source" cambia solo a "Red Hat CDN".

9. **Installation Source.** No tocar. Con la ISO DVD dice "Auto-detected installation media" (`sr0`). Done si se entró.

10. **Software Selection.** Base Environment: seleccionar **"Server"** (el radio button, no "Server with GUI" que viene marcado por defecto). En "Additional software for Selected Environment" no marcar nada. Done.

    **Salida esperada:** el tablero muestra "Server" bajo Software Selection.

    Qué observar: "Server with GUI" agrega unos 1 500 paquetes y 5–6 GB, y arranca a un escritorio GNOME. Si alguien lo dejó por error y ya instaló, la corrección es `sudo systemctl set-default multi-user.target` y reiniciar (ver tabla de errores); no hace falta reinstalar.

11. **Security Profile.** No tocar ("Apply security policy: OFF"). En RH134 M5 se comenta qué son estos perfiles.

12. **Root Password.** Escribir la contraseña dos veces. Dejar **sin marcar** "Lock root account" y **sin marcar** "Allow root SSH login with password". Done (si la contraseña es débil pide Done dos veces). **Anotar la contraseña de root**: se usará cuando algo se rompa.

    Qué observar: RHEL 9 prohíbe por defecto que root entre por SSH con contraseña (`PermitRootLogin prohibit-password`). Es la práctica correcta: se entra como usuario normal y se eleva con `sudo`.

13. **User Creation.** Full name `Student`. User name `student`. Marcar **"Make this user administrator"**. Dejar marcado "Require a password to use this account". Contraseña dos veces. Done.

    Qué observar: "Make this user administrator" agrega a `student` al grupo `wheel`, que en RHEL tiene permiso de `sudo` para todo. Si queda sin marcar, `sudo` responderá "student is not in the sudoers file".

14. Revisar que ningún ícono tenga triángulo naranja y pulsar **Begin Installation**.

    **Salida esperada:** barra de progreso con mensajes "Creating xfs on /dev/mapper/rhel-root", "Installing software…", "Installing boot loader", "Performing post-installation setup tasks". Con la ISO DVD tarda 8–20 minutos según el disco del equipo.

    Aquí empieza el **descanso**. Nadie toca la VM hasta volver.

15. Al terminar aparece "Complete!" y el botón **Reboot System**. Antes de pulsarlo, retirar la ISO para que la VM arranque del disco:
    - VirtualBox: menú Dispositivos → Unidades ópticas → "Eliminar disco de la unidad virtual" (aceptar "Forzar desmontaje" si lo pide).
    - UTM: ícono de la unidad en la barra de la ventana → CD/DVD → Eject.

    Pulsar Reboot System.

    Qué observar: si la VM vuelve a mostrar el menú "Install Red Hat Enterprise Linux", la ISO sigue conectada: retirarla y en VirtualBox Máquina → Reiniciar; en UTM el botón de reinicio de la barra.

- **Checkpoint:** escribir en el chat "Begin Installation pulsado" con la hora, y al regresar del descanso pegar la línea `Complete!` o el porcentaje en que va.

### Lab 3.2 — Primer arranque, registro y actualización (30 min)

- **Objetivo:** iniciar sesión en la consola, registrar el sistema con la suscripción Developer, comprobar los repositorios y lanzar la primera actualización.

1. **Arranque.** GRUB muestra durante 5 segundos "Red Hat Enterprise Linux (5.14.0-xxx.el9.x86_64) 9.x (Plow)" y arranca. Al final:
   ```text
   Red Hat Enterprise Linux 9.x (Plow)
   Kernel 5.14.0-xxx.el9.x86_64 on an x86_64

   Activate the web console with: systemctl enable --now cockpit.socket

   rhel01 login:
   ```
   Qué observar: el hostname `rhel01` ya aparece en el prompt de login. El mensaje de Cockpit (consola web) es informativo; no se activa en este curso.

2. **Iniciar sesión** como `student`. La contraseña no se muestra al escribirla (ni asteriscos).
   ```text
   rhel01 login: student
   Password:
   Register this system with Red Hat Insights: insights-client --register
   Create an account or view all your systems at https://red.ht/insights-dashboard
   [student@rhel01 ~]$
   ```
   Qué observar: el prompt `[student@rhel01 ~]$` dice usuario, host, directorio actual (`~` = home) y `$` = usuario normal. Se detalla en el Bloque 5.

3. **Registrar el sistema.** Pide el usuario de Red Hat (el login, no el correo) y la contraseña. La primera vez que se usa `sudo` aparece una advertencia y pide la contraseña de `student`.
   ```bash
   sudo subscription-manager register
   ```
   **Salida esperada:**
   ```text
   We trust you have received the usual lecture from the local System
   Administrator. It usually boils down to these three things:
       #1) Respect the privacy of others.
       #2) Think before you type.
       #3) With great power comes great responsibility.

   [sudo] password for student:
   Registering to: subscription.rhsm.redhat.com:443/subscription
   Username: <usuario-redhat>
   Password:
   The system has been registered with ID: 6f1c2a9e-....-....
   The registered system name is: rhel01
   ```
   Qué observar: no hace falta `attach`. Si aparece "Invalid username or password", revisar que se está usando el nombre de usuario de Red Hat y que los términos de la Developer Subscription se aceptaron en el portal.

4. **Estado de la suscripción.**
   ```bash
   sudo subscription-manager status
   ```
   **Salida esperada:**
   ```text
   +-------------------------------------------+
      System Status Details
   +-------------------------------------------+
   Overall Status: Disabled
   Content Access Mode is set to Simple Content Access. This host has access to content, regardless of subscription status.

   System Purpose Status: Disabled
   ```
   Qué observar: "Overall Status: Disabled" **no es un error**. Con Simple Content Access el estado de suscripción individual está deshabilitado porque el acceso lo da la cuenta; la frase clave es "This host has access to content".

5. **Identidad registrada.**
   ```bash
   sudo subscription-manager identity
   ```
   **Salida esperada:**
   ```text
   system identity: 6f1c2a9e-....-....
   name: rhel01
   org name: Nombre Apellido
   org ID: 1234567
   ```
   Qué observar: `org ID` es el número de la organización de la cuenta Developer y `system identity` es el UUID con el que el portal identifica **esta** VM. Conviene anotarlo: si la VM se borra sin ejecutar `subscription-manager unregister`, el sistema se da de baja desde el portal buscando ese UUID (la cuenta Developer admite hasta 16 sistemas y se llena rápido reinstalando).

6. **Repositorios disponibles.** Usar siempre `sudo` con `dnf`, incluso para consultar.
   ```bash
   sudo dnf repolist
   ```
   **Salida esperada:**
   ```text
   Updating Subscription Management repositories.
   repo id                              repo name
   rhel-9-for-x86_64-appstream-rpms     Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
   rhel-9-for-x86_64-baseos-rpms        Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
   ```
   Qué observar: **BaseOS** contiene el sistema base (kernel, systemd, bash); **AppStream** las aplicaciones y lenguajes (httpd, podman, python). En UTM los repos dicen `aarch64`. Sin registro, esta salida sería vacía con el mensaje "This system is not registered with an entitlement server".

7. **Prueba rápida de instalación** (confirma que los repos funcionan; `tree` se usará el Día 2).
   ```bash
   sudo dnf -y install tree
   ```
   **Salida esperada (resumida):**
   ```text
   Updating Subscription Management repositories.
   Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)        12 MB/s |  45 MB     00:03
   Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)     15 MB/s |  55 MB     00:03
   Dependencies resolved.
   ================================================================================
    Package     Architecture   Version             Repository                       Size
   ================================================================================
   Installing:
    tree        x86_64         1.8.0-10.el9        rhel-9-for-x86_64-baseos-rpms    56 k
   ...
   Installed:
     tree-1.8.0-10.el9.x86_64
   Complete!
   ```
   Qué observar: la primera vez descarga los metadatos de los repos (unos 100 MB); las siguientes son inmediatas.

8. **Lanzar la actualización completa** y dejarla corriendo en la consola. Se seguirá trabajando por SSH desde el Bloque 4 mientras termina.
   ```bash
   sudo dnf -y update
   ```
   **Salida esperada (resumida):** lista de decenas o cientos de paquetes "Upgrading:", descarga de 300 MB–1 GB, "Running transaction", y al final `Complete!`. Puede tardar 5–20 minutos.

   Qué observar: si la lista incluye `kernel`, el kernel nuevo se usará solo tras reiniciar; se hará antes del snapshot. Alternativa de registro que también conecta con Red Hat Insights: `sudo rhc connect` (pide usuario y contraseña; muestra "Connected to Red Hat Subscription Management" y "Connected to Red Hat Insights"). En el curso se usa `subscription-manager` porque muestra mejor qué hace cada paso.

- **Checkpoint:** pegar en el chat la salida de `sudo dnf repolist` (las dos líneas de repos).

---

## Bloque 4 — Acceso por SSH desde el equipo propio

### Conceptos (5 min)

**SSH** (Secure Shell) es el protocolo con el que se administra cualquier servidor Linux: una terminal remota cifrada por el puerto 22/TCP. El servidor (`sshd`) ya está instalado y **activo** en RHEL 9 "Server" desde el primer arranque, y el firewall (`firewalld`, también activo desde el primer arranque) ya trae permitido el servicio `ssh` en la zona por defecto: por eso hoy no hay que abrir ningún puerto. Ese es justamente el criterio de Red Hat: lo único abierto de fábrica es la puerta por la que se administra. El cliente lo traen Windows 10/11 (PowerShell) y macOS (Terminal).

**Port forwarding en una frase:** la VM está en una red NAT privada (10.0.2.x) que el equipo del participante no ve; con la regla creada en el Lab 2.2, el hipervisor escucha en el puerto 2222 del equipo y reenvía todo lo que llega al puerto 22 de la VM. Por eso se conecta a `localhost:2222` y se llega a `rhel01:22`. Se profundiza el Día 5.

**Huella (fingerprint):** la primera vez que un cliente se conecta a un servidor SSH, este presenta su clave pública y el cliente muestra un resumen (huella) preguntando si se confía. Al aceptar, queda guardada en `~/.ssh/known_hosts` del equipo del participante; en conexiones futuras SSH compara y avisa si el servidor cambió (lo que pasa si se reinstala la VM). Es el mecanismo que evita que alguien se haga pasar por el servidor.

Ventaja práctica de pasar a SSH ahora: en la terminal del propio equipo se puede copiar y pegar, cambiar el tamaño de letra y desplazar la pantalla, cosas que la consola de la VM no permite.

### Lab 4.1 — Conectarse por SSH a la VM (10 min)

- **Objetivo:** abrir una sesión SSH desde el equipo propio a la VM y reconocerla desde dentro.

1. Abrir la terminal del equipo (Windows: PowerShell o Windows Terminal; Mac: Terminal) y comprobar el cliente.
   ```bash
   ssh -V
   ```
   **Salida esperada:** `OpenSSH_for_Windows_9.5p1, LibreSSL 3.8.2` en Windows o `OpenSSH_9.x, LibreSSL 3.x` en macOS.

   Qué observar: si Windows responde que `ssh` no se reconoce: Configuración → Aplicaciones → Características opcionales → Agregar → "Cliente de OpenSSH". Alternativa: PuTTY con Host `localhost`, Port `2222`.

2. Conectarse.
   ```bash
   ssh -p 2222 student@localhost
   ```
   **Salida esperada:**
   ```text
   The authenticity of host '[localhost]:2222 ([127.0.0.1]:2222)' can't be established.
   ED25519 key fingerprint is SHA256:Ab3dEf9...
   This key is not known by any other names
   Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
   Warning: Permanently added '[localhost]:2222' (ED25519) to the list of known hosts.
   student@localhost's password:
   Register this system with Red Hat Insights: insights-client --register
   Create an account or view all your systems at https://red.ht/insights-dashboard
   Last login: Thu Sep  3 09:12:40 2026 on tty1
   [student@rhel01 ~]$
   ```
   Qué observar: `-p 2222` en minúscula es el puerto para `ssh` (en `scp` es `-P` mayúscula, Día 5). La huella se acepta escribiendo `yes` completo, no `y`. Si la respuesta es un `Connection refused` inmediato, probar `ssh -p 2222 student@127.0.0.1`: la regla de reenvío se ató a la IPv4 `127.0.0.1` y en algunos equipos `localhost` se resuelve primero como `::1` (IPv6), donde el hipervisor no escucha.

3. Confirmar dónde se está y quién está conectado.
   ```bash
   hostname
   w
   ```
   **Salida esperada:**
   ```text
   rhel01
    09:15:02 up 12 min,  2 users,  load average: 0.35, 0.40, 0.22
   USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
   student  tty1     -                09:03    9:12   0.05s  0.03s sudo dnf -y update
   student  pts/0    10.0.2.2         09:14    0.00s  0.03s  0.01s w
   ```
   Qué observar: `tty1` es la sesión de la consola de la VM (la columna WHAT muestra el `dnf update` que sigue corriendo allí; cuando termine dirá `-bash`); `pts/0` es la sesión SSH y viene "desde" `10.0.2.2`, la puerta del NAT: la VM ve al hipervisor, no al equipo real. Ese `2 users` cambia a `1` cuando se cierre la consola.

4. Ejecutar un comando remoto sin abrir sesión interactiva (desde otra ventana del equipo propio, o después de salir):
   ```bash
   ssh -p 2222 student@localhost hostnamectl hostname
   ```
   **Salida esperada:** pide la contraseña y responde `rhel01`.

5. Dejar la sesión SSH abierta: en ella se hace el Bloque 5. Para cerrarla al final del día se usa `exit`.

- **Checkpoint:** pegar en el chat la salida de `w` mostrando la línea `pts/0` con origen `10.0.2.2`.

---

## Bloque 5 — Primer contacto con la shell

### Conceptos (10 min)

**Terminal, consola, shell.** La **consola** es la pantalla física (o la ventana de la VM). La **terminal** es el programa que muestra texto y recibe teclas (la ventana de PowerShell, la sesión SSH). La **shell** es el intérprete que lee lo que se escribe, lo ejecuta y devuelve el resultado; en RHEL es **bash**. Analogía: la terminal es el teléfono; la shell es la persona que contesta.

**Anatomía del prompt.**
```text
[student@rhel01 ~]$
  │       │    │  └── $ = usuario normal.  # = root
  │       │    └───── directorio actual (~ es el home del usuario: /home/student)
  │       └────────── nombre del equipo (hostname)
  └────────────────── usuario con el que se inició sesión
```
Regla de oro que se repetirá todo el curso: si el prompt termina en `#`, cada comando puede destruir el sistema; se trabaja con `$` y se eleva con `sudo` solo lo necesario.

**Estructura de un comando:** `comando -opciones argumentos`. Ejemplo: `ls -la /etc` → comando `ls`, opciones `-l` (lista larga) y `-a` (incluir ocultos), argumento `/etc`. Las opciones cortas se agrupan (`-la`); las largas empiezan con dos guiones (`--help`). Linux distingue mayúsculas de minúsculas: `-r` y `-R` son opciones distintas.

**Dónde está la ayuda.** Tres niveles: `comando --help` (resumen en pantalla), `man comando` (manual completo, paginado) y `/usr/share/doc/` (documentación de cada paquete). Las páginas `man` están organizadas en secciones numeradas; las que importan al administrador: **1** comandos de usuario, **5** formatos de archivos de configuración, **8** comandos de administración. Por eso `man passwd` (sección 1, el comando) y `man 5 passwd` (el archivo `/etc/passwd`) son páginas distintas.

**El árbol de directorios** empieza en `/` (raíz). No hay letras de unidad: todo cuelga de `/`. Hoy solo se reconoce el mapa; el Día 2 se recorre.

### Lab 5.1 — Identidad y estado del sistema (15 min)

- **Objetivo:** responder con comandos las preguntas "¿quién soy, en qué máquina estoy, qué sistema es, cómo está de recursos y tiene red?".

Todos los comandos se ejecutan en la sesión SSH del Lab 4.1.

1. ¿Quién soy?
   ```bash
   whoami
   id
   ```
   **Salida esperada:**
   ```text
   student
   uid=1000(student) gid=1000(student) groups=1000(student),10(wheel) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
   ```
   Qué observar: `uid=1000` es el primer usuario creado; `10(wheel)` confirma que es administrador; el `context=…` es SELinux (Día 7 y RH134 M5).

2. ¿En qué máquina estoy?
   ```bash
   hostname
   hostnamectl
   ```
   **Salida esperada:**
   ```text
   rhel01
    Static hostname: rhel01
          Icon name: computer-vm
            Chassis: vm
         Machine ID: 3c5a…
            Boot ID: 9e1f…
     Virtualization: oracle
   Operating System: Red Hat Enterprise Linux 9.x (Plow)
        CPE OS Name: cpe:/o:redhat:enterprise_linux:9::baseos
             Kernel: Linux 5.14.0-xxx.el9.x86_64
       Architecture: x86-64
    Hardware Vendor: innotek GmbH
     Hardware Model: VirtualBox
   ```
   Qué observar: en la VM del instructor (UTM) se lee `Virtualization: qemu`, `Architecture: arm64`, `Hardware Vendor: QEMU`. Mismo sistema, distinto hardware virtual.

3. ¿Qué sistema y qué kernel?
   ```bash
   cat /etc/os-release
   cat /etc/redhat-release
   uname -r
   uname -m
   ```
   **Salida esperada (resumida):**
   ```text
   NAME="Red Hat Enterprise Linux"
   VERSION="9.x (Plow)"
   ID="rhel"
   ID_LIKE="fedora"
   VERSION_ID="9.x"
   PRETTY_NAME="Red Hat Enterprise Linux 9.x (Plow)"
   ...
   Red Hat Enterprise Linux release 9.x (Plow)
   5.14.0-xxx.el9.x86_64
   x86_64
   ```
   Qué observar: RHEL 9 usa kernel 5.14 con parches retroportados (backports) durante toda su vida; el número `el9` en el nombre del paquete identifica la versión de RHEL. Si `dnf update` instaló un kernel nuevo, `uname -r` mostrará el viejo hasta reiniciar.

4. ¿Desde cuándo está encendido y qué hora tiene?
   ```bash
   uptime
   date
   timedatectl
   ```
   **Salida esperada (resumida):**
   ```text
    09:22:11 up 19 min,  2 users,  load average: 0.52, 0.61, 0.40
   Thu Sep  3 09:22:12 AM EST 2026
                  Local time: Thu 2026-09-03 09:22:13 EST
              Universal time: Thu 2026-09-03 14:22:13 UTC
                    RTC time: Thu 2026-09-03 14:22:13
                   Time zone: America/Panama (EST, -0500)
   System clock synchronized: yes
                 NTP service: active
   ```
   Qué observar: Panamá no usa horario de verano, por eso siempre `EST, -0500`. "NTP service: active" es `chronyd` sincronizando la hora. Si la zona quedó mal: `sudo timedatectl set-timezone America/Panama`.

5. ¿Tengo red?
   ```bash
   ip a
   ping -c 3 redhat.com
   ```
   **Salida esperada (resumida):**
   ```text
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
       inet 127.0.0.1/8 scope host lo
   2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
       link/ether 08:00:27:5e:12:ab brd ff:ff:ff:ff:ff:ff
       inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
          valid_lft 86212sec preferred_lft 86212sec
   ...
   PING redhat.com (34.235.198.240) 56(84) bytes of data.
   64 bytes from ...: icmp_seq=1 ttl=63 time=45.2 ms
   64 bytes from ...: icmp_seq=2 ttl=63 time=44.8 ms
   64 bytes from ...: icmp_seq=3 ttl=63 time=46.1 ms

   --- redhat.com ping statistics ---
   3 packets transmitted, 3 received, 0% packet loss, time 2004ms
   ```
   Qué observar: `-c 3` limita a tres paquetes; sin él `ping` sigue hasta Ctrl+C. Si `ping` falla pero hay Internet (algunos NAT no reenvían ICMP), probar `curl -I https://www.redhat.com` y observar `HTTP/2 200` o `301`. En la VM del instructor (UTM, Emulated VLAN) es esperable que `ping` hacia Internet no responda aunque `curl` y `dnf` funcionen: la red de usuario de QEMU en macOS no reenvía ICMP (⚠️ Verificar en la VM antes de la clase). Forma corta muy útil: `ip -br a`.

6. ¿Cómo están disco, memoria y CPU?
   ```bash
   df -h
   free -m
   lscpu
   lsblk
   ```
   **Salida esperada (resumida):**
   ```text
   Filesystem             Size  Used Avail Use% Mounted on
   devtmpfs               4.0M     0  4.0M   0% /dev
   tmpfs                  1.8G     0  1.8G   0% /dev/shm
   tmpfs                  733M  8.7M  725M   2% /run
   /dev/mapper/rhel-root   15G  2.1G   13G  14% /
   /dev/sda1              960M  270M  691M  29% /boot
   tmpfs                  367M     0  367M   0% /run/user/1000

                  total        used        free      shared  buff/cache   available
   Mem:            3663         300        2900           9         463        3162
   Swap:           3967           0        3967

   Architecture:            x86_64
   CPU(s):                  2
   Model name:              Intel(R) Core(TM) i7-...
   Hypervisor vendor:       KVM
   ...

   NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
   sda             8:0    0   20G  0 disk
   ├─sda1          8:1    0    1G  0 part /boot
   └─sda2          8:2    0   19G  0 part
     ├─rhel-root 253:0    0   15G  0 lvm  /
     └─rhel-swap 253:1    0  3.9G  0 lvm  [SWAP]
   sr0            11:0    1 1024M  0 rom
   ```
   Qué observar: en `lsblk` está el esquema que hizo Anaconda: `sda1` = `/boot`, `sda2` = volumen físico LVM con `rhel-root` y `rhel-swap`. Anaconda dimensiona la swap igual a la RAM detectada cuando esta está entre 2 y 8 GiB: con 4096 MB queda en ~3.9 GiB y `root` con el resto (~15 GiB); con 3072 MB la swap sería ~2.9 GiB (⚠️ Verificar en la VM antes de la clase los tamaños exactos que se van a mostrar). `lscpu` dice `Hypervisor vendor: KVM` porque VirtualBox se presenta ante el kernel con la interfaz de KVM; en UTM la línea puede no aparecer. En UTM el disco es `vda` y hay una partición extra `vda1` montada en `/boot/efi` porque arranca con UEFI. `free -m` muestra menos de 4096 MB porque el kernel reserva parte de la RAM.

- **Checkpoint:** pegar en el chat la salida de:
   ```bash
   echo "$(hostname) $(uname -m) $(uname -r) $(ip -br a | grep -v '^lo' | awk '{print $3}')"
   ```
   **Salida esperada:** `rhel01 x86_64 5.14.0-xxx.el9.x86_64 10.0.2.15/24`.

### Lab 5.2 — Ayuda, atajos, `sudo -i` y navegación inicial (20 min)

- **Objetivo:** consultar `man` y `--help` con soltura, usar Tab, historial y atajos de teclado, elevar privilegios y volver, y moverse por el árbol de directorios.

1. **Manual de un comando.** Abrir el manual de `ls`.
   ```bash
   man ls
   ```
   Dentro del manual: **Espacio** o **PgDn** avanza, **b** retrocede, **/palabra** busca (por ejemplo `/human`), **n** siguiente coincidencia, **N** anterior, **g** inicio, **G** final, **h** ayuda, **q** salir.

   **Salida esperada:** la cabecera `LS(1)  User Commands  LS(1)`, y la búsqueda `/human` lleva a `-h, --human-readable`.

   Qué observar: el `(1)` de la cabecera es la sección. Buscar `-l` con `/ -l` (con espacio antes del guion) evita cientos de coincidencias.

2. **Secciones del manual.** Comparar la página del comando `passwd` con la del archivo `/etc/passwd`.
   ```bash
   man passwd
   man 5 passwd
   man man
   ```
   **Salida esperada:** `PASSWD(1)` describe el comando para cambiar contraseñas; `PASSWD(5)` describe el formato del archivo con sus siete campos separados por `:`; `man man` lista las nueve secciones. Salir de cada una con `q`.

3. **Buscar en los manuales por tema.**
   ```bash
   man -k hostname
   whatis hostnamectl
   ```
   **Salida esperada:**
   ```text
   hostname (1)         - show or set the system's host name
   hostname (5)         - Local hostname configuration file
   hostname (7)         - hostname resolution description
   hostnamectl (1)      - Control the system hostname
   ...
   hostnamectl (1)      - Control the system hostname
   ```
   Qué observar: en una instalación muy reciente `man -k` puede responder `hostname: nothing appropriate.` porque el índice todavía no se generó. Solución: `sudo mandb` (tarda unos segundos) y repetir. `apropos` es sinónimo de `man -k`.

4. **Ayuda corta.**
   ```bash
   ls --help | head -15
   ```
   **Salida esperada:** `Usage: ls [OPTION]... [FILE]...` seguido de la lista de opciones. `--help` sirve para recordar una opción; `man` para entenderla.

5. **Autocompletado con Tab.** Escribir `cat /etc/os-r` y pulsar **Tab**: se completa a `/etc/os-release`. Escribir `cat /etc/host` y pulsar **Tab dos veces**: lista `host.conf hostname hosts`. Escribir `hostn` y Tab: completa `hostname` (y con doble Tab muestra `hostname hostnamectl`).

   Qué observar: Tab evita errores de tipeo y es la forma de descubrir qué hay en un directorio sin salir del comando. Funciona con comandos, rutas y, con `bash-completion`, con opciones.

6. **Historial.**
   ```bash
   history
   ```
   **Salida esperada:** lista numerada de todo lo escrito en la sesión. Con las flechas ↑/↓ se recorren; `!!` repite el último comando; `!25` ejecuta el número 25; **Ctrl+R** y escribir `hostnamectl` busca hacia atrás el último comando que contenga esa palabra (Enter ejecuta, Esc lo deja en la línea para editarlo).

   Ejemplo clásico de `!!`: se intenta leer un archivo que solo root puede ver, aparece el error, y se corrige con:
   ```bash
   cat /etc/shadow
   sudo !!
   ```
   **Salida esperada:**
   ```text
   cat: /etc/shadow: Permission denied
   sudo cat /etc/shadow
   root:$6$....:20700:0:99999:7:::
   ...
   student:$6$....:20700:0:99999:7:::
   ```
   Qué observar: la shell imprime el comando expandido (`sudo cat /etc/shadow`) antes de ejecutarlo. `/etc/shadow` guarda las contraseñas cifradas (Día 3). No se usa `dnf` para este ejemplo porque el `dnf update` de la consola tiene tomado el bloqueo de dnf: cualquier otro `dnf` se quedaría en "Waiting for process with pid … to finish" hasta que termine.

7. **Atajos de control.** Lanzar un comando que tarda y cortarlo.
   ```bash
   sleep 60
   ```
   Pulsar **Ctrl+C**.

   **Salida esperada:** el prompt vuelve de inmediato, mostrando `^C`. Otros atajos: **Ctrl+L** limpia la pantalla (igual que `clear`); **Ctrl+A / Ctrl+E** van al inicio / fin de la línea; **Ctrl+U** borra la línea; **Ctrl+W** borra la última palabra; **Ctrl+D** en una línea vacía cierra la sesión (igual que `exit`), cuidado con él.

8. **Elevar privilegios y volver.**
   ```bash
   sudo -i
   whoami
   pwd
   exit
   whoami
   ```
   **Salida esperada:**
   ```text
   [root@rhel01 ~]# whoami
   root
   [root@rhel01 ~]# pwd
   /root
   [root@rhel01 ~]# exit
   logout
   [student@rhel01 ~]$ whoami
   student
   ```
   Qué observar: `sudo -i` abre una shell de root (prompt `#`, home `/root`) con la contraseña de **student**; `su -` haría lo mismo pero pide la contraseña de **root**. Norma del curso: usar `sudo comando` para tareas puntuales y `sudo -i` solo cuando hay una secuencia larga de administración; siempre `exit` al terminar. `sudo` recuerda la autenticación 5 minutos.

9. **Navegación inicial.**
   ```bash
   pwd
   ls
   ls -la
   cd /etc
   pwd
   ls | head -5
   cd
   pwd
   cd /var/log
   cd ..
   pwd
   cd -
   pwd
   ```
   **Salida esperada (resumida):**
   ```text
   /home/student
   (vacío: no hay archivos visibles todavía)
   total 12
   drwx------. 2 student student  83 Sep  3 09:03 .
   drwxr-xr-x. 3 root    root     21 Sep  3 08:50 ..
   -rw-r--r--. 1 student student  18 Feb  6  2025 .bash_logout
   -rw-r--r--. 1 student student 141 Feb  6  2025 .bash_profile
   -rw-r--r--. 1 student student 492 Feb  6  2025 .bashrc
   /etc
   DIR_COLORS
   DIR_COLORS.lightbgcolor
   GREP_COLORS
   ...
   /home/student
   /var
   /var/log
   ```
   Qué observar: `cd` sin argumento vuelve al home; `..` es el directorio padre; `cd -` regresa al directorio anterior; los archivos que empiezan con `.` están ocultos y solo salen con `-a`. El punto tras los permisos (`drwx------.`) indica que el archivo tiene contexto SELinux; es normal en RHEL.

10. **Mapa de `/`.**
    ```bash
    ls -l /
    tree -L 1 /
    ```
    **Salida esperada (resumida):**
    ```text
    lrwxrwxrwx.   1 root root    7 Feb  6  2025 bin -> usr/bin
    dr-xr-xr-x.   5 root root 4096 Sep  3 08:58 boot
    drwxr-xr-x.  19 root root 3060 Sep  3 09:03 dev
    drwxr-xr-x.  92 root root 8192 Sep  3 09:10 etc
    drwxr-xr-x.   3 root root   21 Sep  3 08:50 home
    lrwxrwxrwx.   1 root root    7 Feb  6  2025 lib -> usr/lib
    lrwxrwxrwx.   1 root root    9 Feb  6  2025 lib64 -> usr/lib64
    drwxr-xr-x.   2 root root    6 Feb  6  2025 media
    drwxr-xr-x.   2 root root    6 Feb  6  2025 mnt
    drwxr-xr-x.   2 root root    6 Feb  6  2025 opt
    dr-xr-xr-x. 220 root root    0 Sep  3 09:02 proc
    dr-xr-x---.   3 root root  163 Sep  3 09:10 root
    drwxr-xr-x.  35 root root 1000 Sep  3 09:14 run
    lrwxrwxrwx.   1 root root    8 Feb  6  2025 sbin -> usr/sbin
    drwxr-xr-x.   2 root root    6 Feb  6  2025 srv
    dr-xr-xr-x.  13 root root    0 Sep  3 09:02 sys
    drwxrwxrwt.   8 root root  172 Sep  3 09:20 tmp
    drwxr-xr-x.  12 root root  144 Sep  3 08:50 usr
    drwxr-xr-x.  20 root root 4096 Sep  3 08:58 var
    ```

    Resumen para la libreta (se profundiza el Día 2):

    | Directorio | Qué contiene |
    |---|---|
    | `/etc` | Configuración del sistema (texto plano) |
    | `/home` | Directorios personales de los usuarios (`/home/student`) |
    | `/root` | Home de root |
    | `/var` | Datos variables: logs (`/var/log`), colas, cachés, sitios web (`/var/www`) |
    | `/tmp` | Temporales; se limpian solos |
    | `/usr` | Programas y bibliotecas instalados (`/usr/bin`, `/usr/sbin`, `/usr/lib64`) |
    | `/boot` | Kernel e imágenes de arranque |
    | `/dev` | Dispositivos (`/dev/sda`, `/dev/null`) |
    | `/proc`, `/sys`, `/run` | Vistas del kernel y estado en ejecución; viven en memoria |
    | `/opt`, `/srv` | Software de terceros; datos servidos (se usará `/srv` el Día 3) |
    | `/mnt`, `/media` | Puntos de montaje manuales y de medios removibles |

    Qué observar: `bin -> usr/bin` y `sbin -> usr/sbin` son enlaces: en RHEL 9 todo el software vive bajo `/usr`. `/tmp` tiene una `t` al final de los permisos (sticky bit, Día 3).

- **Checkpoint:** pegar en el chat la salida de `history | tail -5`.

---

## Bloque 6 — Snapshot `dia01-fin`

### Conceptos (2 min)

El snapshot guarda disco (y opcionalmente memoria) de la VM en un instante. Se toma con la VM apagada para que sea consistente y pequeño. Si al día siguiente algo quedó roto, se restaura `dia01-fin` en un minuto y se sigue la clase.

### Lab 6.1 — Reiniciar, apagar y tomar el snapshot (8 min)

- **Objetivo:** dejar la VM con el kernel actualizado y un snapshot limpio.

Orden del cierre: primero se recorre el checklist del **Reto individual** (puntos 2 a 7) con la VM encendida; después se hace este lab; el punto 8 del checklist se marca al terminar.

1. Comprobar en la consola de la VM que `dnf update` terminó con `Complete!`. Si sigue corriendo, esperar; no apagar en medio de una transacción. Quien no alcance a terminar el update antes del cierre lo deja correr, y hace el reinicio y el snapshot como tarea (el snapshot debe tomarse con el sistema ya actualizado y apagado).
2. Desde la sesión SSH, reiniciar para cargar el kernel nuevo y volver a entrar.
   ```bash
   sudo systemctl reboot
   ```
   **Salida esperada:** la sesión SSH se cierra con `Connection to localhost closed by remote host.` Tras 30–60 s, `ssh -p 2222 student@localhost` vuelve a entrar y `uname -r` muestra un kernel con número mayor que el del Lab 5.1 (si hubo actualización de kernel).
3. Apagar la VM desde la shell.
   ```bash
   sudo systemctl poweroff
   ```
   **Salida esperada:** la ventana de la VM se cierra (VirtualBox) o muestra la VM detenida (UTM). Alternativas equivalentes: `sudo poweroff`, `sudo shutdown -h now`.
4. Tomar el snapshot.
   - **VirtualBox:** en el administrador, seleccionar `rhel01` → menú de la VM (ícono de lista) → **Instantáneas** → botón **Tomar** → Nombre `dia01-fin`, Descripción "RHEL 9 Server instalado, registrado, actualizado, SSH OK" → Aceptar. Para restaurar más adelante: seleccionar la instantánea → **Restaurar**.
   - **UTM:** no tiene gestor de snapshots en la interfaz. Dos opciones:
     - Sencilla: con la VM apagada, clic derecho sobre `rhel01` → **Clone** → renombrar el clon a `rhel01-dia01-fin`. Ocupa espacio en disco pero es a prueba de errores; para "restaurar" se borra la VM rota y se clona de nuevo la copia.
     - Con `qemu-img` (requiere `brew install qemu` en el Mac): con la VM apagada,
       ```bash
       qemu-img snapshot -c dia01-fin ~/Library/Containers/com.utmapp.UTM/Data/Documents/rhel01.utm/Data/*.qcow2
       qemu-img snapshot -l ~/Library/Containers/com.utmapp.UTM/Data/Documents/rhel01.utm/Data/*.qcow2
       ```
       y para restaurar `qemu-img snapshot -a dia01-fin <mismo archivo>`. La ruta del `.utm` depende de cómo se instaló UTM (App Store vs descarga directa) y la VM debe estar completamente apagada, no suspendida (⚠️ Verificar en la VM antes de la clase: ubicar el qcow2 con `ls ~/Library/Containers/com.utmapp.UTM/Data/Documents/rhel01.utm/Data/` y probar crear y listar un snapshot).

   **Resultado esperado:** VirtualBox lista `dia01-fin` bajo "Estado actual"; UTM muestra el clon o `qemu-img snapshot -l` lista `dia01-fin`.

- **Checkpoint:** escribir en el chat "snapshot dia01-fin listo".

---

## Reto individual — Checklist de salida del Día 1 (5 min)

Hoy no hay reto técnico aparte: el reto del día es **terminar la instalación y entrar por SSH**. Se hace **antes** del Lab 6.1, con la VM encendida y desde la sesión SSH: cada participante recorre los puntos 2 a 7 en su VM y marca en el chat cuántos pasa; los puntos 1 y 8 se completan después del snapshot. Quien no pase alguno lo resuelve como tarea con la guía de este documento. Si `dnf update` todavía está corriendo, dejar el punto 6 para el final (cualquier `dnf` esperaría a que termine).

| # | Comprobación | Cómo se verifica |
|---|---|---|
| 1 | La VM arranca sin la ISO y muestra `rhel01 login:` | Encender la VM y observar la consola |
| 2 | Hostname `rhel01` y RHEL 9.x | `hostnamectl` |
| 3 | Instalación "Server" sin GUI | `systemctl get-default` |
| 4 | `student` es administrador y `sudo -i` funciona | `id student` y `sudo -i` seguido de `exit` |
| 5 | La VM tiene IP y salida a Internet | `ip -br a` y `ping -c 3 redhat.com` (o `curl -I https://www.redhat.com` si el NAT no reenvía ICMP) |
| 6 | Sistema registrado con repositorios activos y actualizado | `sudo subscription-manager status`, `sudo dnf repolist`, `sudo dnf history` |
| 7 | Acceso por SSH desde el equipo propio | `ssh -p 2222 student@localhost hostname` desde PowerShell/Terminal |
| 8 | Snapshot `dia01-fin` existe | Pestaña Instantáneas (VirtualBox), clon o `qemu-img snapshot -l` (UTM) |

### Solución (para el instructor)

Salidas que confirman cada punto:

```bash
# 2
hostnamectl | head -2
#  Static hostname: rhel01
#        Icon name: computer-vm
hostnamectl | grep "Operating System"
# Operating System: Red Hat Enterprise Linux 9.x (Plow)

# 3
systemctl get-default
# multi-user.target          <- correcto. Si dice graphical.target, instaló Server with GUI.

# 4
id student
# uid=1000(student) gid=1000(student) groups=1000(student),10(wheel)
sudo -i
# [root@rhel01 ~]#
exit

# 5
ip -br a
# lo       UNKNOWN  127.0.0.1/8 ::1/128
# enp0s3   UP       10.0.2.15/24 fe80::.../64
ping -c 3 redhat.com
# 3 packets transmitted, 3 received, 0% packet loss

# 6
sudo subscription-manager status | grep "Content Access"
# Content Access Mode is set to Simple Content Access. This host has access to content...
sudo dnf repolist
# rhel-9-for-x86_64-appstream-rpms ... / rhel-9-for-x86_64-baseos-rpms ...
sudo dnf history | head -5
# ID | Command line          | Date and time    | Action(s)      | Altered
#  3 | -y update             | 2026-09-03 09:40 | I, U           |  180
#  2 | -y install tree       | 2026-09-03 09:31 | Install        |    1
#  1 |                       | 2026-09-03 08:55 | Install        |  600 EE
# (la transacción 1 es la de Anaconda; su "EE" es normal. "I, U" = instaló paquetes nuevos, como el kernel, y actualizó el resto)

# 7 (desde el equipo del participante)
ssh -p 2222 student@localhost hostname
# rhel01
```

Correcciones rápidas si algo falla, para dictarlas en vivo:

```bash
# Hostname incorrecto
sudo hostnamectl set-hostname rhel01

# Instaló Server with GUI: arrancar en modo texto (no hace falta reinstalar)
sudo systemctl set-default multi-user.target
sudo systemctl reboot

# student no es administrador (ejecutar como root en la consola de la VM: login root)
usermod -aG wheel student
# cerrar sesión y volver a entrar como student para que aplique

# Red no activada en el instalador (sin IP)
nmcli device                     # ver nombre y estado de la interfaz
sudo nmcli connection up enp0s3
sudo nmcli connection modify enp0s3 connection.autoconnect yes

# Zona horaria o teclado incorrectos
sudo timedatectl set-timezone America/Panama
sudo localectl set-keymap latam       # o 'es' (España) / 'us'

# Registro fallido: repetir indicando el usuario y dejar que pida la contraseña
sudo subscription-manager register --username <usuario>
# Evitar --password '<clave>': la contraseña de Red Hat queda en ~/.bash_history y visible en 'ps'
# para cualquier usuario del sistema. Si ya se usó, borrar esa línea con: history -d <número>
# Si el registro quedó a medias: sudo subscription-manager unregister ; sudo subscription-manager clean ; registrar de nuevo
# Si dice "This system is already registered" (se registró desde el instalador con la boot ISO):
# no hay nada que hacer, verificar con sudo subscription-manager identity
```

---

## Cheatsheet del día

| Comando | Para qué sirve |
|---|---|
| `whoami` | Usuario actual |
| `id` | UID, GID y grupos del usuario (buscar `wheel`) |
| `hostname` | Nombre del equipo |
| `hostnamectl` | Nombre, virtualización, SO, kernel y arquitectura en una pantalla |
| `sudo hostnamectl set-hostname rhel01` | Cambiar el hostname |
| `cat /etc/os-release` / `cat /etc/redhat-release` | Versión del sistema: formato estándar / una sola línea |
| `uname -r` / `uname -m` | Versión del kernel / arquitectura |
| `uptime` | Tiempo encendido, usuarios y carga |
| `date` / `timedatectl` | Fecha y hora / zona horaria y estado de NTP |
| `ip a` / `ip -br a` | Interfaces y direcciones IP (completo / resumido) |
| `ping -c 3 destino` | Probar conectividad con tres paquetes |
| `df -h` | Espacio en disco por sistema de archivos |
| `free -m` / `lscpu` | Memoria RAM y swap en MB / información de la CPU |
| `lsblk` | Discos, particiones y volúmenes LVM |
| `man comando` / `man 5 archivo` | Manual completo; sección 5 = archivos de configuración |
| `man -k palabra` / `whatis comando` | Buscar manuales por tema / descripción de una línea |
| `comando --help` | Ayuda breve |
| `history` / `!!` / `Ctrl+R` | Historial / repetir último comando / búsqueda en historial |
| `Tab` / `Ctrl+C` / `Ctrl+L` / `clear` / `exit` | Autocompletar / cancelar el comando en curso / limpiar pantalla (atajo y comando) / cerrar sesión |
| `sudo comando` | Ejecutar un comando como root |
| `sudo -i` … `exit` | Shell de root y regreso |
| `pwd` / `cd` / `cd ..` / `cd -` | Dónde estoy / ir al home / subir / volver al anterior |
| `ls -la` / `ls -l /` / `tree -L 1 /` | Listar con detalle y ocultos / raíz del sistema / árbol de un nivel |
| `sudo subscription-manager register` | Registrar el sistema con la cuenta Red Hat |
| `sudo subscription-manager status` / `identity` | Estado del acceso a contenido / identidad del sistema |
| `sudo dnf repolist` | Repositorios habilitados |
| `sudo dnf -y update` / `sudo dnf -y install paquete` | Actualizar todo el sistema / instalar un paquete |
| `sudo systemctl reboot` / `sudo systemctl poweroff` | Reiniciar / apagar |
| `ssh -p 2222 student@localhost` | Entrar a la VM desde el equipo propio |
| `w` | Quién está conectado y desde dónde |

---

## Notas para el instructor

### Preparar antes de la clase

- **Instalar RHEL 9 dos veces en su propio equipo** con esta guía en la mano, cronometrando: una con la ISO DVD y otra con la boot ISO (para poder acompañar a quien la trajo). Hacer una tercera con "Server with GUI" a propósito y corregirla con `systemctl set-default multi-user.target` para poder demostrarlo sin nervios.
- **Plan B para quien no logre instalar:**
  - **OVA de VirtualBox:** instalar la VM estándar en una máquina x86_64 (o pedirla a un colega), apagarla **sin registrarla** (así cada participante registra con su propia cuenta) y exportarla: Archivo → Exportar servicio virtualizado → formato "Open Virtualization Format 2.0" → `rhel01-dia01.ova` (~2–3 GB). Para importar: Archivo → Importar servicio virtualizado → en "Política de dirección MAC" elegir "Generar nuevas direcciones MAC para todos los adaptadores". Después de importar, revisar el port forwarding: la OVA lo conserva, pero comprobarlo. Si la OVA se exportó registrada, el participante debe ejecutar `sudo subscription-manager unregister && sudo subscription-manager clean` y registrar con su cuenta.
  - **VM exportada de UTM:** con la VM apagada, clic derecho → Share (genera un `.utm` comprimido) o copiar la carpeta `~/Library/Containers/com.utmapp.UTM/Data/Documents/rhel01.utm`. Solo sirve a otro Mac Apple Silicon.
  - Subir ambas a un enlace compartido de la institución (OneDrive/Drive) el día anterior; el descargable pesa varios GB.
- **ISOs a mano:** tener `rhel-9.x-x86_64-dvd.iso`, `rhel-9.x-x86_64-boot.iso` y `rhel-9.x-aarch64-dvd.iso` en un USB o disco externo, y en el mismo enlace compartido, con sus SHA-256 en un archivo de texto. La descarga de 10 GB desde el portal falla o tarda en redes institucionales con proxy.
- Probar la propia cuenta Developer: `sudo subscription-manager register` y `unregister` en la VM de prueba, y confirmar que el portal muestra la suscripción vigente (se renueva cada año).
- Tener abierta la lista de comandos de este archivo para pegarlos en el chat en el momento exacto; los participantes tipean mejor lo que copian.
- Aumentar la letra de la terminal a 16–18 pt y usar fondo claro u oscuro con alto contraste; probar que la ventana de la VM se lee en la pantalla compartida.
- Practicar la tecla Host de VirtualBox (Ctrl derecho) y Control+Option en UTM para no quedarse "atrapado" en la VM frente al grupo.
- **Gestión del tiempo:** cuatro personas instalando a la vez siempre toma más de lo previsto. Mientras Anaconda copia paquetes (paso 14 del Lab 3.1) y mientras corre `dnf update`, aprovechar para completar los conceptos que hayan quedado cortos, responder preguntas y adelantar la explicación de SSH. El descanso se ubica deliberadamente en la copia de paquetes.
- Preparar una respuesta para "¿puedo grabar la sesión?" y para "¿me pasan las diapositivas?": este archivo es el material.
- **Verificar en la VM propia los puntos marcados con ⚠️ en este documento:** tamaños reales de `rhel-root` y `rhel-swap` con 4096 MB de RAM (`lsblk`, `free -m`), si `ping` responde en UTM con Emulated VLAN, el texto con que Anaconda nombra el disco VirtIO, el nombre de la interfaz en UTM (`nmcli device`) la ruta del qcow2 para `qemu-img snapshot`, que la pestaña **Port Forward** aparezca con el modo Emulated VLAN y que `ssh -p 2222 student@localhost` no falle por IPv6. Corregir las salidas esperadas de este archivo con lo observado.

### Qué estudiar si es nuevo en RHEL

1. **`subscription-manager` y Simple Content Access.** Leer `man subscription-manager` (subcomandos `register`, `status`, `identity`, `unregister`, `clean`, `repos --list`). Entender por qué `status` dice "Disabled" con SCA y por qué `attach` ya no aplica. Provocar y leer el error "This system is not registered" con `sudo dnf repolist` antes de registrar.
2. **Anaconda y el particionado automático.** Tras instalar, ejecutar `lsblk`, `sudo pvs`, `sudo vgs`, `sudo lvs` y `cat /etc/fstab` para reconocer `rhel-root` y `rhel-swap`; el Día 6 se construye sobre esto. Probar también el instalador en modo texto (`inst.text`).
3. **Utilidades de systemd para el sistema base:** `hostnamectl`, `timedatectl`, `localectl` (`man hostnamectl`, `localectl list-keymaps | grep -i latam`). Son la forma RHEL 9 de hacer lo que en otras distros se hacía editando archivos.
4. **dnf básico:** `sudo dnf repolist`, `sudo dnf -y update`, `sudo dnf history`, `sudo dnf history info <ID>`. Fijarse en la diferencia BaseOS / AppStream (`man dnf.conf`, sección de opciones de `[repository]`, y `cat /etc/yum.repos.d/redhat.repo`, que genera `subscription-manager`). El Día 6 se profundiza; hoy basta con no sorprenderse por la salida.
5. **Sistema de manuales:** `man man`, `man 1 intro`, `mandb`, `apropos`. En la clase se preguntará "¿y cómo sé qué opción es?"; la respuesta siempre es `man`.
6. Extra: `sudo rhc connect` / `sudo rhc disconnect` y qué es Red Hat Insights, por si alguien lo activó en el instalador (`man rhc`).

### Errores frecuentes de los participantes y cómo resolverlos

| Síntoma | Causa | Solución |
|---|---|---|
| VirtualBox: `VT-x is not available (VERR_VMX_NO_VMX)` o `AMD-V is disabled in the BIOS (VERR_SVM_DISABLED)` | Virtualización deshabilitada en el firmware | Reiniciar, entrar al BIOS/UEFI, activar Intel VT-x / AMD SVM; verificar en Administrador de tareas → CPU → "Virtualización: Habilitado" |
| VirtualBox: `VERR_NEM_VM_CREATE_FAILED`, VM extremadamente lenta, ícono de tortuga | Hyper-V / WSL2 / Docker Desktop tienen tomada la virtualización | Convivir (funciona, lento) o `bcdedit /set hypervisorlaunchtype off` + reinicio; revertir con `auto` cuando necesite WSL2 |
| VirtualBox instaló solo, con usuario `vboxuser` | No se marcó "Omitir instalación desatendida" en el asistente | Borrar la VM y crearla de nuevo marcando la opción |
| El sistema arranca a una pantalla de login gráfica (GNOME) | Se dejó "Server with GUI" en Software Selection | `sudo systemctl set-default multi-user.target && sudo systemctl reboot`; no reinstalar |
| `student is not in the sudoers file. This incident will be reported.` | No se marcó "Make this user administrator" | En la consola, login como `root`, `usermod -aG wheel student`, `exit`, volver a entrar como student |
| `ip a` no muestra IP en `enp0s3`; `ping` dice "Network is unreachable" | La interfaz no se activó en Network & Host Name | `sudo nmcli connection up enp0s3` y `sudo nmcli connection modify enp0s3 connection.autoconnect yes` (ver nombre real con `nmcli device`) |
| "Login incorrect" con la contraseña correcta | Distribución de teclado distinta a la elegida en el instalador (`@`, `-`, `ñ` cambian de lugar) | Probar la contraseña con teclado US en mente; corregir con `sudo localectl set-keymap latam` (o `es`, `us`) tras entrar. Si no se logra entrar, entrar como root y cambiar la contraseña con `passwd student` |
| `subscription-manager register` → "Invalid username or password" | Se usó el correo en vez del Red Hat login, o la cuenta no aceptó los términos Developer | Usar el nombre de usuario; entrar al portal y descargar/aceptar términos; repetir |
| `subscription-manager register` → error de red o timeout | La VM no tiene red o el proxy institucional bloquea `subscription.rhsm.redhat.com` | Verificar `ping -c 2 subscription.rhsm.redhat.com`; en la institución configurar proxy con `subscription-manager config --server.proxy_hostname=... --server.proxy_port=...` |
| `dnf`: "This system is not registered with an entitlement server" y "There are no enabled repositories" | Sistema sin registrar | `sudo subscription-manager register` |
| `dnf`: "Unable to read consumer identity" aunque ya se registró | Se ejecutó `dnf` sin `sudo` (la identidad del sistema en `/etc/pki/consumer/` solo la lee root); es una advertencia, no siempre un error | Usar siempre `sudo dnf …`; si aparece con `sudo`, el registro quedó a medias: `sudo subscription-manager unregister; sudo subscription-manager clean` y registrar de nuevo |
| Tutorial dice `subscription-manager attach --auto` y falla o avisa que está deshabilitado | La cuenta usa Simple Content Access | No hace falta attach; ignorar el paso |
| `subscription-manager status` dice "Overall Status: Disabled" | Comportamiento normal con SCA | Leer la línea "This host has access to content"; no es un error |
| `ssh: connect to host localhost port 2222: Connection refused` | Port forwarding mal escrito (puerto invitado distinto de 22), VM apagada, o regla en otra VM | Revisar la regla en Configuración → Red → Avanzadas → Reenvío de puertos; confirmar que la VM está encendida; en la VM `systemctl status sshd` debe decir `active (running)` |
| `ssh: connect to host localhost port 2222: Connection refused` pero la VM está encendida y `sshd` activo | `localhost` se resolvió como `::1` (IPv6) y la regla de reenvío solo escucha en la IPv4 `127.0.0.1` | Conectar con `ssh -p 2222 student@127.0.0.1`, o dejar vacío el campo "IP anfitrión" de la regla |
| `subscription-manager register` → "This system is already registered" | Se registró desde el instalador (caso de la boot ISO) | No hay nada que hacer: comprobar con `sudo subscription-manager identity`. Para registrar con otra cuenta: `sudo subscription-manager unregister` y volver a registrar |
| El botón "Begin Installation" está gris | Queda una sección con triángulo naranja, casi siempre Installation Destination o User Creation | Entrar a esa sección, pulsar Done aunque no se cambie nada, y volver al tablero |
| El equipo del participante tiene solo 4 u 8 GB de RAM y todo va lentísimo | La VM se llevó demasiada memoria del anfitrión | Apagar la VM y bajar la memoria base a `3072` MB (mínimo del curso); cerrar navegador y Teams/Zoom innecesarios; la VM sin GUI funciona bien con 3 GB |
| `ssh` se queda colgado sin responder ("Connection timed out") | La regla apunta a una IP de anfitrión que no es la del equipo, o firewall del antivirus | Poner "IP anfitrión" en `127.0.0.1` o vacío; probar `ssh -v` para ver dónde se detiene |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | Se reinstaló la VM y la huella del servidor cambió | En el equipo propio: `ssh-keygen -R "[localhost]:2222"` y volver a conectar aceptando la nueva huella |
| `Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password)` | Contraseña o usuario incorrectos (por ejemplo `Student` con mayúscula) | Repetir con `student` en minúscula; si se olvidó la contraseña, cambiarla desde root en la consola |
| La VM vuelve a mostrar "Install Red Hat Enterprise Linux" tras reiniciar | La ISO sigue conectada y arranca antes que el disco | Dispositivos → Unidades ópticas → Eliminar disco (VirtualBox) / Eject (UTM) y reiniciar la VM |
| Pantalla negra tras elegir "Install" (UTM) | El instalador gráfico no arrancó con el display virtual | Reiniciar, en GRUB pulsar `e`, agregar `inst.text` a la línea `linux`, Ctrl+X |
| `man -k algo` → "nothing appropriate" | Índice de manuales no generado aún | `sudo mandb` |
| El mouse y el teclado quedan "atrapados" en la ventana de la VM | Captura de entrada del hipervisor | Ctrl derecho (VirtualBox) / Control+Option (UTM) |
| `dnf update` se interrumpió (cerraron la VM) | Transacción incompleta; pueden quedar paquetes duplicados | `sudo dnf -y update` de nuevo: dnf rehace la transacción sin problema. Si avisa de paquetes duplicados, ejecutar `sudo dnf remove --duplicates` **sin `-y`** y leer la lista antes de aceptar (nunca debe retirar el kernel en uso, el de `uname -r`), y repetir el update. `sudo dnf history` marca la transacción incompleta en la columna Altered |
| `dnf`: `Waiting for process with pid NNNN to finish (running: /usr/bin/dnf -y update)` | Otro dnf en curso (el `dnf update` lanzado en la consola) tiene el bloqueo | Esperar a que termine (o Ctrl+C y repetir después). Solo puede correr un dnf a la vez |

### Diferencias VirtualBox (x86_64) vs UTM (aarch64)

| Aspecto | VirtualBox 7 (participantes) | UTM (instructor) |
|---|---|---|
| ISO | `rhel-9.x-x86_64-dvd.iso` | `rhel-9.x-aarch64-dvd.iso` |
| Disco del sistema | `/dev/sda` (`sda1` = `/boot`, `sda2` = LVM) | `/dev/vda` (`vda1` = `/boot/efi`, `vda2` = `/boot`, `vda3` = LVM) |
| Firmware | BIOS (si no se marcó EFI) | UEFI siempre; por eso existe `/boot/efi` |
| Interfaz de red | `enp0s3` (segundo adaptador: `enp0s8`) | `enp0s1` aprox. (segundo: `enp0s2` aprox.); verificar con `nmcli device` |
| Red NAT | NAT: `10.0.2.15`, gateway `10.0.2.2`, DNS `10.0.2.3`; port forwarding en Avanzadas; `ping` a Internet funciona | Emulated VLAN: mismos valores `10.0.2.x`; port forwarding en pestaña Port Forward; `ping` a Internet suele **no** responder (sin ICMP), usar `curl`. "Shared Network" da `192.168.64.x` sin port forwarding |
| `uname -m` / `hostnamectl` | `x86_64` / `Architecture: x86-64`, `Virtualization: oracle` | `aarch64` / `Architecture: arm64`, `Virtualization: qemu` |
| Nombre de paquetes | `…el9.x86_64` | `…el9.aarch64` |
| Repos | `rhel-9-for-x86_64-baseos-rpms` | `rhel-9-for-aarch64-baseos-rpms` |
| Snapshots | Nativos (pestaña Instantáneas) | Clonar la VM apagada o `qemu-img snapshot` sobre el qcow2 |
| Liberar teclado/mouse | Ctrl derecho | Control+Option |
| Segunda consola (tty2) | Host+F2 | Sin atajo directo; usar SSH |

Decir en voz alta cada vez que se vea una de estas diferencias en pantalla: "en su VirtualBox esto se llama `sda`/`enp0s3`". Repetirlo evita que alguien copie `vda` el Día 6.

### Preguntas probables y respuesta corta

- **¿Por qué no Ubuntu, que es más popular?** Porque el objetivo es administrar servidores institucionales, donde predomina RHEL y sus derivados, y porque la certificación RHCSA es sobre RHEL. Lo aprendido se traslada a Ubuntu con cambios menores (`apt` en vez de `dnf`, `netplan` en vez de `nmcli`).
- **¿Puedo usar Rocky o Alma en lugar de RHEL para practicar en casa?** Sí; todo es idéntico salvo `subscription-manager` (no existe) y algunos nombres de repos. Para el examen y para la institución conviene practicar con RHEL.
- **¿Qué pasa cuando vence la suscripción Developer?** El sistema sigue funcionando pero `dnf` deja de tener acceso al contenido. Se renueva gratis cada año aceptando los términos en el portal y volviendo a registrar si hace falta.
- **¿Puedo usar la suscripción Developer en los servidores de la institución?** No: es para uso individual de desarrollo y pruebas. La institución necesita suscripciones institucionales (o usar Rocky/Alma sin soporte, con las implicaciones de cumplimiento vistas hoy).
- **¿Puedo instalar la interfaz gráfica después?** Sí: `sudo dnf group install "Server with GUI"` y `sudo systemctl set-default graphical.target`. En el curso no se hace; los servidores no la llevan.
- **¿Por qué en inglés?** Porque los mensajes de error se buscan en inglés; la documentación, foros y el examen están en inglés.
- **¿Cuál es la diferencia entre root y sudo?** root es la cuenta con todos los privilegios; `sudo` permite a un usuario normal ejecutar comandos como root dejando registro de quién hizo qué (`/var/log/secure`, Día 4). Trabajar como root todo el tiempo borra esa trazabilidad y multiplica el daño de un error.
- **¿Qué es "Plow"?** El nombre en clave de RHEL 9 (RHEL 8 fue "Ootpa", RHEL 10 es "Coughlan").
- **¿Por qué se creó swap si tengo 4 GB de RAM?** Es espacio de intercambio en disco: red de seguridad cuando la RAM se agota y requisito para hibernación. El tamaño lo decide Anaconda según la RAM. Día 6.
- **¿Cómo salgo de `man` / `less`?** Con `q`. Es la pregunta más frecuente del primer día.
- **¿Cómo cambio el teclado o la hora ahora que ya instalé?** `sudo localectl set-keymap latam`, `sudo timedatectl set-timezone America/Panama`.
- **¿RHEL 9 o RHEL 10 para el examen?** El EX200 se rinde sobre la versión que Red Hat tenga vigente en el momento de la inscripción (hoy RHEL 9; RHEL 10 se va incorporando). Los objetivos y las herramientas son prácticamente los mismos, así que practicar en RHEL 9 sirve para cualquiera de los dos. ⚠️ Verificar la versión vigente en la página oficial del EX200 antes de inscribirse.
- **¿Qué es Cockpit, lo del mensaje de login?** Una consola web de administración (puerto 9090). No se activa en el curso, pero es útil en la institución; se menciona el Día 4.

### Relación con el examen RHCSA

El examen EX200 no evalúa instalar el sistema (las VMs vienen instaladas), pero este día construye el entorno y toca directamente estos objetivos:

- **Understand and use essential tools:** "Access a shell prompt and issue commands with correct syntax"; "Locate, read, and use system documentation including man, info, and files in /usr/share/doc".
- **Operate running systems:** "Boot, reboot, and shut down a system normally" (`systemctl reboot`, `systemctl poweroff`); "Log in and switch users in multiuser targets" (`sudo -i`, `exit`; `su -` se practica el Día 3).
- **Deploy, configure, and maintain systems:** el registro con `subscription-manager` es la base de "Install and update software packages from Red Hat Network, a remote repository, or from the local file system" (Día 6). También `systemctl get-default` / `set-default` ("Configure systems to boot into a specific target automatically").
- **Manage basic networking:** se anticipa `nmcli device`, `ip a`, `hostnamectl set-hostname` (Día 5).
- En el examen se trabaja siempre por SSH o consola sin GUI: la soltura con `man`, Tab y el historial que se empieza a formar hoy ahorra minutos que en el examen valen puntos.

---

## Tarea y preparación para el día siguiente

1. **Quien no terminó la instalación:** completarla con los Labs 3.1 y 3.2 de esta guía (o importar la OVA / VM exportada del instructor) y pasar las 8 comprobaciones del checklist. Enviar al instructor la salida de `hostnamectl` y de `sudo dnf repolist` antes de la próxima clase.
2. **Todos: 20 minutos de práctica con `man` y navegación.** Entrar por SSH y:
   - Leer `man 1 intro` completo (es corto) y `man hostnamectl`.
   - Con `man ls` encontrar qué hacen `-h`, `-t`, `-r` y `-S`; probarlas sobre `/etc` (`ls -lhS /etc | head`).
   - Recorrer con `cd` y `ls -la`: `/etc`, `/var/log`, `/usr/bin`, `/home`, `/root` (observar el "Permission denied" en `/root` y explicarlo con lo visto hoy).
   - Practicar Ctrl+R para recuperar `subscription-manager status` del historial y ejecutarlo con `sudo`.
   - Cerrar con `exit`.
3. **Snapshot `dia01-fin` tomado** con la VM apagada. Si durante la práctica algo se rompe, restaurarlo y avisar en el chat qué pasó: es material para la clase.
4. **Anotar y guardar** las contraseñas de root y de student, y el nombre de usuario de Red Hat. Se necesitarán todo el curso.
5. **No agregar todavía** el segundo adaptador de red ni los discos adicionales: se añaden el Día 5 (adaptador Host-only y discos de 5 GB para el Día 6), con instrucciones propias.
6. Opcional (Windows): instalar Windows Terminal desde la Microsoft Store para tener pestañas y copiar/pegar cómodo en las sesiones SSH.
7. Lectura previa para el Día 2 (10 minutos): en la VM, `man 7 hier` describe el árbol de directorios que se recorrerá; no hace falta memorizarlo.
