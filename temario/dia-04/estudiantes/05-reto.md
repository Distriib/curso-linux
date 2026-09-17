# Reto — Tres tickets

**Solos, sin ayuda.** Antes de empezar, pegar en la terminal la línea que mando por el chat: prepara los tres problemas en tu VM. Si no se termina en clase, queda de tarea.

> **Mesa de ayuda — servidor `rhel01`**
> Por cada ticket, entregar: qué estaba mal, cómo lo encontraste, cómo lo corregiste.

## Ticket #2026-0401 — "El servidor está lento"

> Los usuarios reportan lentitud desde hace unos minutos. El monitoreo muestra la CPU al 50 % sostenido. Identificar el proceso responsable: **PID**, **usuario** que lo ejecuta y **programa real** que está corriendo (el nombre que muestra puede engañar). Terminarlo de forma ordenada y borrar el archivo que lo originó. Explicar cómo se distingue de un proceso legítimo del kernel.

## Ticket #2026-0402 — "El monitor dejó de registrar"

> `monitor.service` no registra nada desde hace un rato. Dejarlo **activo y habilitado** para que arranque con el sistema, y explicar qué estado tenía la unidad y qué significa.

## Ticket #2026-0403 — "Software de origen no autorizado"

> Auditoría detectó el programa `/usr/bin/htop`, que no está en la lista de software aprobado. Indicar: **qué paquete** lo instaló, **desde qué repositorio**, y **en qué transacción** de `dnf` (número y comando). Después **deshacer exactamente esa transacción** (no desinstalar a mano) y comprobar que el programa ya no existe.

## Verificación

Pegar en el chat la salida completa de:

```bash
pgrep -a kworkerd
ls /tmp/kworkerd
systemctl is-active monitor
systemctl is-enabled monitor
journalctl -u monitor -n 1 --no-pager
rpm -q htop
dnf history | head -3
```

Está bien si: `pgrep` no imprime nada; `ls` dice `No such file or directory`; `active`; `enabled`; una línea de `monitor` con hora de hace menos de un minuto; `package htop is not installed`; la transacción más reciente dice `history undo`.
