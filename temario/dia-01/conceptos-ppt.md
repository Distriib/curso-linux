# Día 1 — Bloque de conceptos (25 min)

Guion para la presentación. Cada lámina trae **En pantalla** (lo que se proyecta,
corto) y **Qué decir** (lo que explica el instructor).

---

## Lámina 1 — Portada

**En pantalla:**
- Red Hat System Administration I y II
- 40 horas · 10 jornadas · Red Hat Enterprise Linux 9
- Procuraduría General de la Nación

**Qué decir:**
Presentación propia y ronda rápida del grupo: nombre, qué administran hoy,
si han tocado Linux antes. Reglas: cámara si se puede, preguntas en cualquier
momento, y los "checkpoints" donde todos pegan la salida de un comando en el
chat para confirmar que nadie quedó atrás.

---

## Lámina 2 — Qué es Linux (en sentido estricto)

**En pantalla:**
- Linux = el **kernel**
- Arranca primero, habla con el hardware
- Reparte CPU, memoria, disco y red
- Linus Torvalds, 1991

**Qué decir:**
Cuando decimos "Linux" solemos referirnos al sistema completo, pero en sentido
estricto Linux es solo el núcleo: el programa que arranca primero y hace de
intermediario entre el hardware y todo lo demás. Un kernel solo no le sirve a
nadie: no tiene ni siquiera un comando para listar archivos.

---

## Lámina 3 — GNU/Linux

**En pantalla:**
- Kernel Linux + herramientas **GNU**
- `ls`, `cp`, `grep`, bash, compilador, bibliotecas
- Proyecto GNU: 1983, Richard Stallman
- Por eso el nombre correcto es **GNU/Linux**

**Qué decir:**
Todo lo que uno escribe en la terminal viene mayoritariamente del proyecto GNU,
que es anterior a Linux. Se juntaron y formaron un sistema completo. En una sala
de servidores nadie dice "GNU/Linux", dice "Linux", y está bien; pero conviene
saber de dónde viene cada pieza.

---

## Lámina 4 — Qué es una distribución

**En pantalla:**
- Kernel + herramientas + gestor de paquetes + instalador + soporte
- El kernel es el **motor**; la distro es el **auto completo**
- Debian/Ubuntu → `apt`
- Red Hat/Fedora → `dnf`
- SUSE → `zypper`

**Qué decir:**
Todas usan el mismo kernel; cambia la carrocería. Lo que realmente distingue a
una distribución empresarial es el gestor de paquetes, la política de
actualizaciones y si alguien responde el teléfono cuando algo falla.

---

## Lámina 5 — La familia Red Hat

**En pantalla:**

```
Fedora  →  CentOS Stream  →  RHEL  →  Rocky / AlmaLinux
```

- **Fedora** — laboratorio, versión nueva cada 6 meses
- **CentOS Stream** — vista previa de la próxima RHEL
- **RHEL** — el producto: estable, certificado, 10 años
- **Rocky / Alma** — reconstrucciones gratuitas, sin soporte

**Qué decir:**
Lo que funciona en Fedora llega a RHEL dos o tres años después, ya probado.
Punto importante para ellos: el CentOS Linux que muchos conocieron **ya no
existe**; CentOS 7 quedó sin parches el 30 de junio de 2024. Si en la
institución hay servidores CentOS 7, están sin parches de seguridad. Rocky y
Alma nacieron en 2021 para llenar ese hueco, y todo lo del curso funciona
igual en ellas salvo el registro de suscripción.

---

## Lámina 6 — Ciclo de vida de RHEL 9

**En pantalla:**

| Fase | Hasta |
|---|---|
| Full Support | mayo 2027 |
| Maintenance Support | mayo 2032 |
| Extended Life Cycle (pago) | ~2035 |

- RHEL 9 salió en mayo de 2022
- Una versión menor cada 6 meses: 9.0 … 9.8

**Qué decir:**
Diez años de soporte es la razón por la que una institución paga por RHEL en
lugar de usar algo gratis. "Soporte" significa tres cosas concretas:
repositorios con parches firmados y trazables (cada corrección tiene su CVE),
derecho a abrir casos con ingenieros de Red Hat, y certificación de hardware y
software de terceros. Para una entidad pública, el primero y el tercero son los
que sostienen una auditoría.

---

## Lámina 7 — Qué es una suscripción

**En pantalla:**
- RHEL no se compra: se **suscribe**
- Sin suscripción el sistema funciona, pero `dnf` no baja nada
- Nosotros usamos la **Developer Subscription**: gratis, hasta 16 sistemas
- Solo desarrollo y pruebas, no producción

**Qué decir:**
La suscripción da acceso a los repositorios y al soporte por un año renovable.
Un detalle que les va a ahorrar dolores de cabeza con tutoriales viejos: hoy las
cuentas usan **Simple Content Access**. Antes había que registrar el sistema y
además "adjuntar" una suscripción. Hoy basta con registrar. Si un tutorial dice
`subscription-manager attach --auto`, ese paso ya no aplica.

---

## Lámina 8 — Arquitectura de un sistema Linux

**En pantalla:**

```
┌─────────────────────────────────────────┐
│ Aplicaciones y shell: bash, vim, dnf…   │  ← aquí trabajamos
├─────────────────────────────────────────┤
│ systemd (PID 1) y servicios             │
├─────────────────────────────────────────┤
│ Kernel: procesos, memoria, red, SELinux │
├─────────────────────────────────────────┤
│ Hardware (real o virtual)               │
└─────────────────────────────────────────┘
```

**Qué decir:**
El usuario nunca habla con el kernel directamente. Habla con la shell, que
interpreta lo que escribe y lanza programas; esos programas le piden cosas al
kernel. **systemd** es el primer proceso que arranca el kernel, el PID 1, y es
quien inicia todo lo demás: red, SSH, hora, firewall. Cuando en el Día 4 usemos
`systemctl`, estaremos hablando con esta capa.

---

## Lámina 9 — Máquina virtual e hipervisor

**En pantalla:**
- **VM**: una computadora simulada por software, con su CPU, RAM, disco y red
- **Hipervisor tipo 1** (sobre el hardware): VMware ESXi, Hyper-V, KVM
- **Hipervisor tipo 2** (sobre tu escritorio): VirtualBox, UTM, VMware Fusion
- Nosotros usamos tipo 2; los conceptos son idénticos

**Qué decir:**
El sistema operativo de adentro cree que está en una máquina real. Eso es lo que
nos permite romper cosas sin consecuencias. En producción usarían tipo 1, pero
todo lo que administren aquí se administra igual allá.

---

## Lámina 10 — ISO y snapshot

**En pantalla:**
- **ISO**: la imagen de un DVD de instalación en un solo archivo
- Se "inserta" en la unidad óptica virtual y la VM arranca desde ella
- **Snapshot**: una foto del estado completo de la VM
- Permite romper el sistema y volver atrás en segundos

**Qué decir:**
Vamos a tomar un snapshot al final de cada jornada. Es la red de seguridad del
curso: si algo queda roto, se restaura el del día anterior y se sigue. Insistir
en esto, porque el que no lo hace lo lamenta en el Día 6 o el Día 10.

---

## Lámina 11 — ¿Y RHEL 10?

**En pantalla:**
- Salió en mayo de 2025 (kernel 6.12)
- Exige procesadores modernos (x86-64-v3)
- Novedades: image mode, criptografía post-cuántica, DNF 5
- **Todo lo del curso funciona igual en la 10**

**Qué decir:**
Usamos la 9 porque es lo que hay en producción, su documentación está madura y
el examen RHCSA se rinde sobre ella. Pero `dnf`, `systemd`, `firewalld`,
SELinux, LVM y Podman son idénticos en la 10. Un punto que sí les conviene
anotar: RHEL 10 exige procesadores posteriores a 2013 aproximadamente, así que
si piensan migrar, lo primero es revisar el inventario de servidores.

---

## Lámina 12 — Qué vamos a hacer hoy

**En pantalla:**
1. Crear la máquina virtual
2. Instalar RHEL 9 sin escritorio
3. Registrarla y actualizarla
4. Conectarse por SSH
5. Primeros comandos
6. Snapshot

**Qué decir:**
Al final del día cada uno se va con un servidor Red Hat funcionando, registrado
y accesible desde su propia máquina. Ese servidor es el que vamos a usar las
40 horas: lo que construyamos hoy sigue vivo el Día 10.

---

## Notas de tiempo

- 25 minutos para 12 láminas: unos 2 minutos por lámina.
- Si el reloj aprieta, las láminas 6 (ciclo de vida) y 11 (RHEL 10) se retoman
  después, mientras el instalador copia paquetes.
- Si alguien llega sin la ISO descargada, que la ponga a bajar **ahora**,
  mientras escucha este bloque.
