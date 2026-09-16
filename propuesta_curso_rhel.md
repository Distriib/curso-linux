# Programa de Capacitación — Red Hat System Administration I y II

**Procuraduría General de la Nación de Panamá — Dirección de Informática**

Solicitud de bienes y servicios N° 085 del 16 de junio de 2026

| Dato del programa | Detalle |
| --- | --- |
| Curso | Red Hat System Administration I (RH124) y II (RH134) |
| Modalidad | Virtual, en vivo, con instructor |
| Idioma | Español |
| Duración total | 40 horas |
| Jornadas | 10 |
| Horas por jornada | 4 |
| Plataforma técnica | Red Hat Enterprise Linux 9 sobre máquina virtual individual |
| Participantes | José Quiróz, Gustavo Rivera, Rogelio Villalaz y Juan Basmeson |
| Certificación asociada | Red Hat Certified System Administrator (RHCSA, examen EX200) |

## Enfoque del programa

El programa es práctico. Cada participante instala y administra su propia máquina virtual con Red Hat Enterprise Linux 9 durante las 40 horas. Los conceptos se exponen de forma breve y se aplican de inmediato sobre ese servidor. Los laboratorios acumulan estado: lo construido en una jornada se usa en la siguiente. Cada jornada cierra con un ejercicio individual de verificación. El temario sigue la ficha técnica solicitada y orienta la preparación del examen RHCSA.

<div style="page-break-after: always;"></div>

## Estructura del programa

| Jornada | Tema | Módulos de la ficha | Horas |
| --- | --- | --- | --- |
| 1 | Instalación de RHEL 9 y fundamentos de la shell | RH124 M1, M4, M5, M6 | 4 |
| 2 | Archivos, texto y respaldos desde la terminal | RH124 M1, M2 | 4 |
| 3 | Usuarios, grupos, sudo y permisos | RH124 M3, M4 | 4 |
| 4 | Procesos, systemd, logs y gestión de software | RH124 M4, M6; RH134 M2, M8 | 4 |
| 5 | Redes, DNS, SSH y diagnóstico | RH124 M5; RH134 M3 | 4 |
| 6 | Almacenamiento, LVM y Stratis | RH124 M7; RH134 M6 | 4 |
| 7 | Automatización con Bash y tareas programadas | RH134 M1, M2 | 4 |
| 8 | Seguridad: firewalld, SELinux y hardening | RH124 M8; RH134 M5 | 4 |
| 9 | Servicios de red y contenedores con Podman | RH134 M4, M5, M6, M7 | 4 |
| 10 | Diagnóstico, recuperación y reto integrador | RH134 M8; integración RH124 y RH134 | 4 |

## Metodología

- Concepto expuesto en bloques breves, siempre antes de la práctica.
- Práctica guiada simultánea: instructor y participantes ejecutan a la vez.
- Puntos de control por bloque para confirmar que nadie queda atrás.
- Reto individual al cierre, con formato de ticket de soporte.
- Revisión de resultados, dudas y encuadre de la jornada siguiente.

## Recursos para el participante

- Máquina virtual propia con RHEL 9, construida en la primera jornada y conservada tras el curso.
- Material escrito de cada jornada en formato digital.
- Laboratorios documentados paso a paso y reproducibles.
- Guía de comandos por jornada, para repaso hacia el examen RHCSA.

<div style="page-break-after: always;"></div>

### Jornada 1 — Instalación de RHEL 9 y fundamentos de la shell

**Objetivo.** Instalar, registrar y actualizar un servidor RHEL 9 sin entorno gráfico, y administrarlo por SSH desde la shell.

**Contenidos:**
- Linux, kernel y distribuciones; familia Red Hat: Fedora, CentOS Stream, RHEL, Rocky.
- Ciclo de vida de RHEL 9, suscripciones y Simple Content Access.
- Arquitectura por capas: hardware virtual, kernel, systemd y espacio de usuario.
- Virtualización tipo 2: hipervisor, ISO, snapshots, red NAT, reenvío de puertos.
- Instalación con Anaconda: perfil Server, particionado LVM, usuario administrador.
- Registro de suscripción, repositorios BaseOS y AppStream, actualización con DNF.
- Shell Bash y SSH: prompt, manuales, historial, sudo, navegación del árbol.

**Laboratorios:**
- **Creación de la máquina virtual.** Máquina verificada con 2 vCPU, 4 GB, disco de 20 GB y red NAT.
- **Instalación con Anaconda.** Servidor sin entorno gráfico, con particionado LVM y usuario administrador.
- **Registro, actualización y acceso por SSH.** Sistema registrado, actualizado y administrable por SSH desde el equipo del participante.
- **Primera sesión y snapshot.** Inventario de kernel, red, disco y memoria, con snapshot guardado.

**Ejercicio de cierre.** Checklist de ocho comprobaciones sobre instalación, privilegios, red, registro y acceso SSH.

**Módulos de la ficha:** RH124 M1; RH124 M4, M5 y M6 (parcial).

### Jornada 2 — La terminal a fondo: archivos, texto y respaldos

**Objetivo.** Gestionar archivos y enlaces desde la línea de comandos, analizar logs con filtros de texto, respaldar y editar configuraciones.

**Contenidos:**
- Shell Bash: comandos, rutas, variables de entorno, PATH, alias, historial, atajos.
- Jerarquía de directorios de RHEL 9 y función de cada rama.
- Archivos: listado, copia, movimiento, borrado, metadatos, enlaces duros y simbólicos.
- Texto: grep con expresiones regulares, cut, sort, uniq, wc, diff.
- Redirección, tuberías, tee, xargs y los tres flujos estándar.
- Búsquedas avanzadas con find y locate: criterios y acciones.
- Empaquetado y compresión con tar, gzip, bzip2, xz y zip; edición con vim.

**Laboratorios:**
- **Recorrido por la shell y el árbol.** Variables, alias e historial inspeccionados y directorios de primer nivel recorridos.
- **Estructura de trabajo.** Árbol de documentos, clientes, respaldos y logs, con enlaces verificados por inodo.
- **Análisis de log y búsqueda con find.** Consultas resueltas sobre un log de 300 líneas y archivos localizados por tamaño y fecha.
- **Respaldo, restauración y edición.** Respaldos comprimidos restaurados con verificación de fidelidad y una configuración editada en vim.

**Ejercicio de cierre.** Ticket individual: analizar un log de 200 líneas, ordenar por usuario y empaquetar el respaldo con fecha.

**Módulos de la ficha:** RH124 M2; RH124 M1 (refuerzo).

<div style="page-break-after: always;"></div>

### Jornada 3 — Usuarios, grupos, sudo y permisos

**Objetivo.** Administrar cuentas y grupos, delegar privilegios acotados con sudo y controlar el acceso a datos compartidos con permisos y ACL.

**Contenidos:**
- Identidad: UID y GID, bases de datos de cuentas, grupos primarios y suplementarios.
- Ciclo de vida de cuentas: alta, modificación, contraseñas y caducidad.
- Bloqueo, expiración y baja de cuentas; archivos huérfanos tras la eliminación.
- Privilegios: su frente a sudo, sudoers, visudo y grupo wheel.
- Permisos estándar, notación octal y simbólica, umask y propiedad.
- Permisos especiales setuid, setgid y sticky bit en directorios compartidos.
- ACL con setfacl y getfacl; auditoría del uso de sudo.

**Laboratorios:**
- **Radiografía de las cuentas.** Bases de datos de cuentas interpretadas, con identidades y grupos documentados.
- **Estructura institucional.** Tres grupos de área y cuatro usuarios con UID fijos y caducidad de 90 días.
- **Delegación limitada con sudo.** Regla acotada para reiniciar servicios concretos, verificada y rastreada en los logs.
- **Carpetas compartidas con ACL.** Dos carpetas con herencia de grupo, un buzón con sticky bit y auditoría de solo lectura.

**Ejercicio de cierre.** Ticket de alta de proyecto: usuarios con caducidad, carpeta con herencia, auditor de solo lectura y sudo acotado.

**Módulos de la ficha:** RH124 M3; RH124 M4 (parcial).

### Jornada 4 — Procesos, systemd, logs y gestión de software

**Objetivo.** Controlar procesos y servicios con systemd, interpretar los logs e instalar software desde repositorios oficiales, de terceros y locales.

**Contenidos:**
- Procesos: identificadores, árbol, estados, señales, trabajos, prioridades, carga, memoria.
- systemd: unidades, activo frente a habilitado, enmascarado, targets y unidades propias.
- journald y rsyslog: filtros, archivos de log, journal persistente y rotación.
- Hora del sistema con chronyd, base para correlacionar registros.
- RPM: consultas a la base de paquetes, dependencias y firmas GPG.
- DNF: búsqueda, instalación, historial, deshacer, grupos y módulos.
- Repositorios: BaseOS, AppStream, CodeReady Builder, EPEL y repositorio local desde ISO.

**Laboratorios:**
- **Control de procesos.** Procesos que saturan la CPU, localizados, repriorizados y terminados de forma controlada.
- **Servicio web y unidad propia.** Apache habilitado al arranque y una unidad de systemd que se reinicia tras un fallo.
- **Trazabilidad de logs y rotación.** Mensajes desviados a un archivo propio, journal persistente y rotación diaria.
- **Repositorios EPEL y local desde ISO.** EPEL con verificación de firma e ISO montada como repositorio para instalar sin internet.

**Ejercicio de cierre.** Tres tickets: un proceso disfrazado que satura la CPU, un servicio enmascarado y un paquete no autorizado.

**Módulos de la ficha:** RH124 M4 y M6; RH134 M2 y M8 (parcial).

<div style="page-break-after: always;"></div>

### Jornada 5 — Redes: NetworkManager, DNS, SSH y diagnóstico

**Objetivo.** Configurar direccionamiento estático persistente, resolución de nombres y acceso por clave SSH, y diagnosticar la red por capas.

**Contenidos:**
- IPv4 e IPv6: máscara, CIDR, gateway, rutas, puertos y modos de red del hipervisor.
- Inspección del sistema: interfaces, rutas, sockets y pruebas de alcance.
- Resolución de nombres: consultas DNS, archivo de equipos y orden de resolución.
- NetworkManager: perfiles, IP estática, DNS, rutas, MTU, nombre de equipo.
- SSH: configuración del servicio, claves ed25519, agente, alias, hosts conocidos.
- Transferencia con scp, sftp y rsync; túneles locales sobre SSH.
- Troubleshooting por capas, con tabla de síntoma, capa y herramienta.

**Laboratorios:**
- **Inventario de red y nombres.** Inventario de la red del servidor y origen determinado para cada nombre resuelto.
- **Perfil de red estático.** Adaptador host-only con IP estática persistente, DNS propio y acceso por SSH desde el anfitrión.
- **Claves SSH, transferencia y túnel.** Acceso sin contraseña con claves y alias, rsync entre directorios y un puerto alcanzado por túnel.
- **Diagnóstico por capas con tcpdump.** Caída de interfaz provocada, localizada por capas y confirmada con el tráfico ICMP y DNS.

**Ejercicio de cierre.** Ticket con fallas de red aleatorias: diagnóstico por capas, corrección persistente y documentación de cada hallazgo.

**Módulos de la ficha:** RH124 M5; RH134 M3.

### Jornada 6 — Almacenamiento: particiones, sistemas de archivos, LVM y Stratis

**Objetivo.** Integrar discos nuevos, montarlos de forma persistente y ampliar su capacidad en caliente con LVM sin interrumpir el servicio.

**Contenidos:**
- Cadena disco, partición, sistema de archivos y punto de montaje; tablas MBR y GPT.
- Particionado con parted y fdisk; sistemas de archivos XFS, ext4 y vfat.
- Montaje persistente por UUID y etiquetas en fstab; salida del modo de emergencia.
- Memoria de intercambio: swap en partición y en archivo, con prioridades.
- LVM: volúmenes físicos, grupos, volúmenes lógicos, extensión en caliente, snapshots.
- Stratis con aprovisionamiento ligero y VDO: deduplicación y compresión transparentes.
- Automontaje bajo demanda con autofs y unidades de systemd.

**Laboratorios:**
- **Particionado y montaje persistente.** Dos discos en GPT, con sistemas XFS y ext4 montados por UUID tras reiniciar.
- **Swap en partición y en archivo.** Partición y archivo de intercambio persistentes que suman memoria tras el arranque.
- **Volumen LVM ampliado en caliente.** Volumen lógico montado y ampliado con otro disco, sin detener a los usuarios.
- **Pool Stratis con snapshot.** Pool montado de forma persistente, con snapshot tomado y capacidad ampliada con un segundo disco.

**Ejercicio de cierre.** Ticket de servicio: volumen ext4 de respaldos persistente, ampliado en caliente, más un archivo de intercambio.

**Módulos de la ficha:** RH124 M7; RH134 M6 (parcial).

<div style="page-break-after: always;"></div>

### Jornada 7 — Automatización con Bash y programación de tareas

**Objetivo.** Automatizar tareas con scripts de Bash y programarlos con cron, at y temporizadores de systemd, sin intervención humana.

**Contenidos:**
- Bash: shebang, variables, argumentos, códigos de salida, modo estricto, depuración.
- Condicionales, bucles, funciones, arrays y control de flujo.
- Procesamiento de texto con awk, sed, cut, sort, uniq, tr y xargs.
- Scripts de respaldo, altas masivas de usuarios, verificación de salud y monitoreo de disco.
- Tareas programadas con cron, anacron, at, batch y temporizadores de systemd.
- Archivos temporales y registro de la actividad en el journal.
- Automatización a escala: infraestructura como código, Ansible, Cockpit, Satellite, Insights.

**Laboratorios:**
- **Scripts base con validación.** Scripts ejecutables desde cualquier directorio, que validan argumentos y devuelven códigos de salida diferenciados.
- **Respaldo y procesamiento de logs.** Respaldos fechados con retención de siete días y registro, más estadísticas extraídas de un log.
- **Utilidades de administración.** Alta masiva de usuarios desde CSV, verificación de salud y alerta de uso de disco.
- **Tareas programadas y escala.** Entradas de crontab, un trabajo con at, un temporizador persistente y un playbook de Ansible idempotente.

**Ejercicio de cierre.** Script validado que comprime y elimina logs por antigüedad, deja registro y queda programado a las 02:00.

**Módulos de la ficha:** RH134 M1 y M2.

### Jornada 8 — Seguridad: firewalld, SELinux y hardening

**Objetivo.** Publicar servicios con zonas de firewalld, corregir denegaciones de SELinux sin desactivarlo y endurecer el acceso al servidor.

**Contenidos:**
- firewalld sobre nftables: zonas, servicios, puertos, runtime frente a permanente.
- Clasificación por origen e interfaz, reglas enriquecidas con registro y modo pánico.
- SELinux: control discrecional frente a obligatorio, modos, contextos, tipos, política targeted.
- Puertos, contextos de archivo y booleanos; reetiquetado frente a cambio puntual.
- Denegaciones: mensajes AVC, búsqueda en auditoría, informes y permisivo por dominio.
- Endurecimiento de SSH, calidad de contraseñas, bloqueo por intentos fallidos, caducidad.
- Superficie de ataque, actualizaciones automáticas de seguridad, auditoría con OpenSCAP.

**Laboratorios:**
- **Publicación de Apache con firewalld.** Portal accesible por ambas rutas de red, con reglas permanentes y zona por origen.
- **Corrección de denegaciones de SELinux.** Servidor web con error 403 recuperado en un puerto no estándar, reetiquetando y registrando el puerto.
- **Directorio propio de publicación.** Sitio publicado desde un directorio del participante con contexto persistente, y una denegación resuelta.
- **Endurecimiento del acceso.** SSH solo con llaves, sin root y con banner legal, más bloqueo por intentos y parches automáticos.

**Ejercicio de cierre.** Ticket individual: restaurar una web caída corrigiendo puerto SELinux, contexto de archivos y zona de firewalld, sin desactivar ninguno.

**Módulos de la ficha:** RH124 M8; RH134 M5.

<div style="page-break-after: always;"></div>

### Jornada 9 — Servicios de red y contenedores con Podman

**Objetivo.** Publicar servicios web, NFS, SMB y FTP, y desplegar contenedores rootless que arrancan con el sistema mediante systemd.

**Contenidos:**
- Apache: hosts virtuales por nombre, índices de directorio y HTTPS con mod_ssl.
- NFS y autofs: exportaciones, montaje permanente, mapas indirecto y directo.
- Samba: recursos compartidos por grupo, cuentas propias y clientes de acceso.
- FTP con vsftpd y usuarios enjaulados; criterios para migrar a SFTP o HTTPS.
- Podman rootless: imágenes UBI, registros, volúmenes etiquetados, Containerfile, skopeo.
- Contenedores como servicios: Quadlet, systemd de usuario y persistencia sin sesión.
- SELinux y firewalld por servicio: contextos, booleanos y zonas.

**Laboratorios:**
- **Host virtual y HTTPS.** Dos sitios en la misma dirección IP, con índice de directorios, cifrado TLS y contextos correctos.
- **Compartición NFS, SMB y FTP.** Exportación NFS montada al usarse, carpeta Samba con credenciales y FTP funcional.
- **Contenedores rootless con Podman.** Contenedor web que sirve contenido del host y responde fuera de la máquina virtual.
- **Imagen propia gestionada por systemd.** Imagen construida a partir de UBI y contenedor registrado como servicio que sobrevive al reinicio.

**Ejercicio de cierre.** Ticket: desplegar un contenedor de portal autoarrancable en un puerto definido, con su directorio exportado por NFS y automontado.

**Módulos de la ficha:** RH134 M4 y M7; RH134 M6 (parcial); RH134 M5 (refuerzo).

### Jornada 10 — Diagnóstico, recuperación del sistema y reto integrador

**Objetivo.** Diagnosticar fallas por capas, recuperar sistemas que no arrancan y resolver incidentes reales verificando cada corrección.

**Contenidos:**
- Método de diagnóstico en siete pasos y revisión ordenada por capas.
- Herramientas por capa: servicios, journal, kernel, sockets, archivos abiertos, disco, auditoría, trazas.
- journald a fondo: arranques previos, filtros por campo, formatos, persistencia, retención.
- rsyslog: facilidades, prioridades, plantillas, envío remoto; rotación y auditoría con auditd.
- Proceso de arranque, GRUB2 y targets de rescate y emergencia.
- Recuperación: contraseña de root desde el initramfs, fstab dañado, kernel e initramfs.
- Integración: Apache, LVM, NFS, Podman, cron, firewalld y SELinux en un servidor.

**Laboratorios:**
- **Radiografía de un servidor sano.** Capas de diagnóstico recorridas y espacio liberado de un archivo aún abierto.
- **Centralización y auditoría de logs.** Logs enviados y recibidos por TCP con reglas propias, rotación y auditoría de cuentas.
- **Recuperación de un sistema caído.** Contraseña de root restablecida desde el initramfs y arranque recuperado tras un fstab inválido.
- **Servidor integrador.** Sitio Apache, exportación NFS, respaldos sobre LVM y contenedor rootless, validados automáticamente.

**Ejercicio de cierre.** Resolver sin pistas ocho tickets sobre un servidor averiado, documentando causa, corrección persistente y verificación externa.

**Módulos de la ficha:** RH134 M8; RH124 M4 (parcial); integración de RH124 M3, M5, M7 y M8 y de RH134 M1 a M7.

<div style="page-break-after: always;"></div>

## Cobertura de la ficha técnica

| Módulo | Contenido de la ficha | Jornada(s) | Tratamiento |
| --- | --- | --- | --- |
| RH124 M1 | Conceptos básicos, arquitectura Linux, shell y terminal, navegación | 1, 2 | Laboratorio |
| RH124 M2 | Crear, copiar y mover archivos; compresión y empaquetado; búsquedas avanzadas | 2 | Laboratorio |
| RH124 M3 | Creación de usuarios, administración de grupos, permisos estándar y especiales | 3, 10 | Laboratorio |
| RH124 M4 | Procesos, servicios, logs del sistema | 1, 3, 4, 10 | Laboratorio |
| RH124 M5 | Configuración IP, DNS, SSH | 1, 5, 10 | Laboratorio |
| RH124 M6 | RPM, DNF/YUM, repositorios | 1, 4 | Laboratorio |
| RH124 M7 | Particiones, sistemas de archivos, montaje de discos | 6, 10 | Laboratorio |
| RH124 M8 | Firewall básico, introducción a SELinux, buenas prácticas | 8, 10 | Laboratorio |
| RH134 M1 | Bash scripting, variables, loops, automatización de tareas | 7, 10 | Laboratorio |
| RH134 M2 | Cron, at, automatización empresarial | 4, 7, 10 | Laboratorio y demostración |
| RH134 M3 | NetworkManager, configuración avanzada IP, troubleshooting | 5 | Laboratorio |
| RH134 M4 | HTTP, NFS, SMB, FTP | 9 | Laboratorio |
| RH134 M5 | SELinux avanzado, firewalld, hardening básico | 8, 9 | Laboratorio y demostración |
| RH134 M6 | LVM, Stratis, VDO, automontaje | 6, 9 | Laboratorio y demostración |
| RH134 M7 | Introducción a Podman, gestión de imágenes, contenedores Linux | 9 | Laboratorio |
| RH134 M8 | Diagnóstico de fallos, logs avanzados, recuperación básica | 4, 10 | Laboratorio |

Los puntos marcados como demostración se presentan en pantalla, con explicación del instructor y material de referencia escrito, sin práctica individual. Corresponden a temas de alcance amplio —Ansible y herramientas de gestión, auditoría con OpenSCAP y almacenamiento con VDO— cuya práctica completa excede las 40 horas contratadas. El resto se ejecuta como laboratorio sobre la máquina virtual de cada participante.

<div style="page-break-after: always;"></div>

## Condiciones de entrega

- Modalidad virtual en vivo, con instructor, en idioma español.
- Cuarenta horas distribuidas en diez jornadas de cuatro horas.
- Material de cada jornada entregado en formato digital a los cuatro participantes.
- Laboratorios documentados paso a paso y reproducibles después del curso.
- Un ejercicio individual de verificación al cierre de cada jornada.
- Contenido alineado con los objetivos del examen de certificación RHCSA (EX200).
- Máquina virtual con Red Hat Enterprise Linux 9 por participante, conservada al finalizar.

## Requisitos del participante

- Equipo con al menos 8 GB de memoria RAM y 40 GB de espacio libre en disco.
- Virtualización habilitada en el firmware y software de virtualización instalado.
- Conexión a internet estable, con acceso a los repositorios de Red Hat.
- Cuenta gratuita de Red Hat Developer para registrar la suscripción del laboratorio.
- Privilegios de administración sobre el equipo propio.

## Resultado esperado

Al finalizar las diez jornadas, cada participante administra un servidor Red Hat Enterprise Linux 9 que construyó desde cero. Instala el sistema, gestiona usuarios y permisos, controla servicios y logs, configura la red, integra almacenamiento, automatiza tareas, aplica seguridad con SELinux y firewalld, publica servicios de red y contenedores, y recupera un sistema que no arranca. El material y los laboratorios quedan disponibles para el repaso posterior y para la preparación del examen RHCSA.
