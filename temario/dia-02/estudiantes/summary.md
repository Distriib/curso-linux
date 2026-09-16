# Día 2 — La terminal a fondo

## Qué vamos a hacer

0. **Repaso e instalaciones** — Confirmar acceso por SSH y repositorios activos, instalar las herramientas del día
1. **Cómo funciona la shell**  — Comandos, rutas, variables, historial y atajos
2. **El árbol de RHEL 9**  — Qué guarda cada carpeta del sistema
3. **Archivos y enlaces**  — Crear, copiar, mover, borrar, permisos a simple vista
4. **Analizar texto**  — `grep`, `sort`, `uniq`, `cut`, tuberías y redirección
5. **Buscar** — `find` y `locate` para encontrar cualquier archivo
6. **Respaldos**  — Comprimir con `tar`, restaurar y verificar
7. **Editar con vim** — Lo mínimo para sobrevivir: entrar, escribir, guardar, salir

## Cómo corre el día

| Bloque | Min | Qué |
|---|---:|---|
| 0 | 5 | Repaso y herramientas que usaremos |

| 1 | 20 | Cómo funciona la shell y el árbol de RHEL (explicado en consola) |

| 2 | 15 | **Concepto:** `ls -l`, copiar, borrar, enlaces (explicado en consola) |
| 2 | 30 | **Lab 2.1:** construir `~/empresa` — por partes, solos, solución de cada parte en consola |

| 3 | 15 | **Concepto:** grep, tuberías, redirección (explicado en consola) |
| — | 15 | Descanso |
| 3 | 30 | **Lab 3.1:** analizar un log de 300 líneas — por partes, solos, solución de cada parte en consola |

| 4 | 10 | **Concepto:** `find` y `locate` (explicado en consola) |
| 4 | 25 | **Lab 4.1:** buscar con `find` — por partes, solos, solución de cada parte en consola |

| 5 | 10 | **Concepto:** respaldos con `tar` (explicado en consola) |
| 5 | 25 | **Lab 5.1:** respaldar `~/empresa` y `/etc` — por partes, solos, solución de cada parte en consola |

| 6 | 5 | **Concepto:** vim, dos estados y seis teclas (explicado en consola) |
| 6 | 20 | **Lab 6.1:** editar una configuración con vim — por partes, teclas dictadas |

| — | 0 | Reto individual — solo si sobra tiempo; si no, es tarea |
| — | 10 | Cierre y snapshot |
| — | 5 | Colchón (margen para imprevistos) |

---

## Al terminar

Moverse por el servidor sin interfaz gráfica, encontrar cualquier archivo,
analizar un log de 300 líneas y respaldar una carpeta.

---

## Tarea

1. **Snapshot `dia02-fin`** con la VM apagada. Si algo quedó roto, restaurar
   `dia01-fin` y repetir los labs 2.1 y 3.1.
2. **Hacer el reto** (Ticket #2026-0142) si no se hizo en clase, y mandar la salida de la entrega por el chat antes de la próxima clase.
3. **`vimtutor`** (20 min): completar al menos las lecciones 1 a 4.
4. **Práctica de 20 minutos** (sin mirar el material):
   - Crear `~/practica/{a,b,c}` con un archivo en cada uno, empaquetarlos en
     `/tmp/practica.tar.gz` y restaurarlos en `/tmp/r`.
   - `sudo find /var/log -mmin -60` y explicar en una línea qué archivos
     aparecen y por qué.
   - Extraer de `/etc/passwd` los usuarios con shell `/bin/bash` (`grep`,
     `cut`), ordenados.
   - Abrir `~/empresa/documentos/app.conf` con vim, cambiar `puerto=8080` por
     `puerto=9090` con `:%s`, guardar.
   - `grep -Ev "^#|^$" /etc/ssh/ssh_config` y contar las líneas.
5. **Escribí tu propio resumen en markdown** (15 min): con `vim`, creá
   `~/notas-dia2.md` con al menos un título (`#`), un subtítulo (`##`), una
   lista de lo que aprendiste hoy, y un bloque de código de tres backticks
   (\`\`\`) con el comando que más te costó.
6. **Lectura** (10 min): `man 7 hier` y `man tar` (la sección de ejemplos al
   inicio).
7. **Para mañana** (usuarios, grupos y permisos), 10 min de investigar solos:
   - `ls -l /etc/hostname /etc/shadow ~/empresa/documentos/informe1.txt` — ¿qué significan las 9 letras de `rwxr-xr-x`? ¿Por qué `shadow` tiene todas en `-`?
   - `id` y `id root` — ¿qué es el `uid`, el `gid`, y por qué `student` está en el grupo `wheel`?
   - `man 5 passwd` — los 7 campos que vieron hoy, explicados.