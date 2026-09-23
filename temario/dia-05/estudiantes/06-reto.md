# Reto — Ticket #RED-05

**Solos, sin ayuda.** Si no se termina en clase, queda de tarea.

> "Desde esta mañana el servidor `rhel01` tiene problemas de red. Algunos usuarios dicen que no pueden entrar por la IP del laboratorio; otros, que el servidor no resuelve nombres. No sabemos qué se tocó. Déjelo como estaba y documente qué encontró."

## Preparar

El instructor comparte en el chat un bloque de comandos que **provoca las fallas**. Copiarlo, pegarlo, y a partir de ahí no volver a mirarlo.

## Lo que hay que hacer

1. Recorrer **las capas en orden**, como en el bloque anterior. Anotar, por cada falla, **qué comando la delató** y qué salida vio.
2. Corregir con `nmcli`. No con `ip addr` ni `ip route`: esos cambios se pierden al reiniciar y el ticket pide dejarlo **como estaba**.
3. Verificar que todo volvió:
   - `ping -c 2 8.8.8.8`
   - `getent hosts redhat.com`
   - desde su computadora: `ssh rhel01 hostname`

## Entrega

Una tabla en el chat, una fila por falla:

| Falla | Comando que la delató | Comando de corrección |
|---|---|---|
| | | |

Más la salida de las tres verificaciones.

---

**Pista única:** son dos fallas, y las dos están en perfiles de NetworkManager. Ninguna está en el hipervisor ni en el firewall.
