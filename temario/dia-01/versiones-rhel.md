# RHEL 7, 8, 9 y 10 — qué cambió

Para responder si preguntan. No es material de clase.

---

| | Salió | Soporte hasta | Kernel |
|---|---|---|---|
| **RHEL 7** | 2014 | **Terminó en junio de 2024** | 3.10 |
| **RHEL 8** | 2019 | Mayo 2029 | 4.18 |
| **RHEL 9** | 2022 | Mayo 2032 | 5.14 |
| **RHEL 10** | 2025 | ~2035 | 6.12 |

---

## Lo que cambió en cada una

**RHEL 7** — Llegó `systemd`, que reemplazó al arranque clásico de Unix. Fue el
cambio más grande de la historia de RHEL y todavía hay gente resentida. También
llegaron `firewalld` y XFS como sistema de archivos por defecto.

**RHEL 8** — `dnf` reemplazó a `yum`. Los repositorios se dividieron en BaseOS y
AppStream. Podman reemplazó a Docker. El firewall pasó a usar nftables por debajo.

**RHEL 9** — La configuración de red dejó los archivos `ifcfg-*` y pasó a
*keyfiles*. OpenSSL 3 empezó a rechazar certificados con SHA-1. Root ya no puede
entrar por SSH con contraseña.

**RHEL 10** — Exige procesadores posteriores a 2013. Trae criptografía
post-cuántica y el modo imagen (el sistema operativo desplegado como contenedor).
DNF 5. Se eliminó el servidor gráfico Xorg.

---

## Lo que hay que saber decir

**Si preguntan por CentOS 7:** quedó sin parches de seguridad el 30 de junio de
2024, junto con RHEL 7. Si tienen servidores así, están expuestos. Es la
pregunta más útil que les podés devolver.

**Si preguntan por qué no usamos la 10:** porque la 9 es lo que hay en
producción, su documentación está madura y el examen RHCSA se rinde sobre ella.
Y porque `dnf`, `systemd`, `firewalld`, SELinux, LVM y Podman son idénticos en
las dos.

**Si preguntan si conviene migrar:** de RHEL 7 sí, urgente, está sin soporte.
De 8 a 9 hay tiempo hasta 2029. A la 10, primero hay que revisar si los
procesadores la aguantan.
