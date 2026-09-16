# Día 1 — Instalar RHEL 9 y primeros pasos

**4 horas**

## Qué vamos a hacer

1. **Conceptos** (25 min) — Qué es Linux, la familia Red Hat, qué es una máquina virtual
2. **Crear la VM** (30 min) — 2 vCPU, 4 GB RAM, 20 GB disco, red NAT
3. **Instalar RHEL 9** (45 min) — Perfil *Server* sin escritorio
4. **Registrar y actualizar** (30 min) — Suscripción Red Hat y `dnf update`
5. **Conectarse por SSH** (15 min) — Administrar el servidor desde su propia máquina
6. **Primeros comandos** (45 min) — Identidad, sistema, red, ayuda y navegación
7. **Snapshot** (15 min) — Guardar el estado para poder volver atrás

## Cómo corre el día

| Hora | Min | Qué |
|---|---:|---|
| 0:00–0:10 | 10 | Apertura: presentación, reglas, sondeo de quién trae todo listo |
| 0:10–0:35 | 25 | **Concepto:** Linux, la familia Red Hat, suscripciones, qué es una VM |
| 0:35–0:45 | 10 | **Lab 2.1:** verificar ISO, virtualización y cuenta |
| 0:45–1:05 | 20 | **Lab 2.2:** crear la máquina virtual `rhel01` |
| 1:05–1:50 | 45 | **Lab 3.1:** instalar RHEL 9 con Anaconda |
| 1:50–2:05 | 15 | Descanso (la instalación copia paquetes mientras tanto) |
| 2:05–2:35 | 30 | **Lab 3.2:** primer arranque, registro de la suscripción y actualización |
| 2:35–2:50 | 15 | **Lab 4.1:** conectarse por SSH desde el equipo propio |
| 2:50–3:35 | 45 | **Labs 5.1 y 5.2:** primeros comandos, ayuda, atajos y navegación |
| 3:35–3:50 | 15 | Reto: checklist de salida. **Lab 6.1:** reinicio, apagado y snapshot |
| 3:50–4:00 | 10 | Cierre, cheatsheet y tarea |

---

## Al terminar

Un servidor Red Hat Enterprise Linux 9 funcionando, registrado, actualizado
y accesible por SSH. Y saber moverse en la terminal.

## Lo que necesitan traer

- Cuenta en developers.redhat.com
- ISO de RHEL 9.8 descargada
- VirtualBox instalado (Windows) o UTM (Mac)
- Virtualización habilitada en la BIOS
