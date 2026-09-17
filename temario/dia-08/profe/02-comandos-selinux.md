# Comandos — SELinux: qué es (guía del instructor)

Quince minutos. Apache sigue caído desde el Lab 1.2: aprovechalo, es la motivación. Tres cosas que tienen que quedar: (1) SELinux es una segunda capa de permisos que limita a los **procesos** según su **tipo**, y a root también; (2) tres modos, y `Disabled` nunca; (3) la etiqueta viaja con `mv` y se corrige con `restorecon`. El resto se practica.

**Qué es DAC:** Discretionary Access Control, "control a discreción del dueño": los permisos `rwx` del Día 3. El dueño del archivo decide quién lee y escribe, y root puede todo.
**Qué es MAC:** Mandatory Access Control, "control obligatorio": una política del sistema que nadie puede saltarse, ni root. Dice qué puede hacer cada proceso. SELinux es la implementación de MAC en RHEL.
**Qué es SELinux:** Security-Enhanced Linux. Viene activado en toda instalación de RHEL. Es lo que ponía el punto después de los permisos en `ls -l` (Días 2 y 3) y el `context=` en `id` (Día 1).
**Qué es la política:** el conjunto de reglas "el tipo X puede hacer Y sobre el tipo Z". La escribe Red Hat; nosotros solo la ajustamos con `semanage` (Bloque 3). La de RHEL se llama `targeted`: confina servicios ("blancos"), no a las personas.
**Qué es un proceso comprometido:** un programa (Apache) que un atacante logró controlar por una vulnerabilidad. Ese programa ahora hace lo que el atacante quiere, con los permisos del usuario con el que corría.
**Qué es el usuario `apache`:** la cuenta de sistema (Día 3) con la que corren los procesos de Apache después de arrancar como root.
**Qué es explotar:** aprovechar una vulnerabilidad para ejecutar código en el servidor. Vos lo sabés de sobra; para ellos, una frase: "engañar al programa para que haga algo que no debía".
**Qué es un modo:** cómo se comporta SELinux: `Enforcing` aplica y registra; `Permissive` solo registra; `Disabled` no carga política.
**Qué es `getenforce`:** muestra el modo actual (una palabra).
**Qué es `setenforce 0` / `1`:** cambia a Permissive / Enforcing hasta el próximo reinicio. Con `sudo`.
**Qué es `sestatus`:** el estado completo: modo actual, modo del archivo de configuración, nombre de la política.
**Qué es `/etc/selinux/config`:** el archivo que fija el modo con el que **arranca** el sistema. Hoy solo se mira. `SELINUX=disabled` es el pecado capital: al volver a habilitarlo hay que reetiquetar todo el disco (minutos).
**Qué es un contexto o etiqueta:** los cuatro campos `usuario:rol:tipo:nivel` que tiene todo archivo, proceso y puerto. Usar la palabra "etiqueta" en clase; "contexto" es el nombre técnico.
**Qué es el tipo:** el tercer campo, el que termina en `_t`. Es lo único que decide el acceso en la política `targeted`. Los otros tres campos se nombran una vez y se ignoran.
**Qué es `system_u` / `unconfined_u`:** el primer campo, el usuario SELinux. `system_u` para lo que crea el sistema, `unconfined_u` para lo que crea una persona. No decide el acceso: por eso el mismo archivo puede tener uno u otro y funcionar igual.
**Qué es `object_r`:** el rol de los archivos (objetos). Siempre igual. Ignorar.
**Qué es `s0`:** el nivel de seguridad. En `targeted` es siempre `s0`. El `s0-s0:c0.c1023` de tu sesión es lo mismo con rango. Ignorar.
**Qué es un dominio:** el tipo de un **proceso** (`httpd_t`). "Tipo" para archivos y puertos, "dominio" para procesos; es la misma idea.
**Qué es confinado / `unconfined_t`:** un proceso confinado tiene un dominio con reglas (Apache). Tu shell es `unconfined_t`: sin reglas, SELinux no te limita. Por eso vos podés leer `/root/index.html` y Apache no.
**Qué es `ls -Z`:** `ls` mostrando la etiqueta. `-Zd` (con `-d`, como `ls -ld` del Día 2) muestra la de la carpeta misma y no la de su contenido.
**Qué es `ps -eZ`:** `ps -e` (todos los procesos, Día 4) con la etiqueta de cada uno.
**Qué es `id -Z`:** la etiqueta de tu sesión.
**Qué es heredar:** un archivo nuevo toma el tipo de la carpeta donde nace. Un archivo en `/var/www/html` nace `httpd_sys_content_t`; uno en `/root` nace `admin_home_t`.
**Qué es un inodo:** el registro interno del archivo en el disco (Día 2). `mv` dentro del mismo disco cambia el nombre y la ubicación pero es **el mismo inodo**: se lleva su etiqueta. `cp` crea un inodo nuevo en el destino: nace con la etiqueta del destino.
**Qué es `restorecon`:** "restore context": le pone al archivo la etiqueta que la política dice que debería tener según su ruta. `-v` muestra lo que cambió; `-R` recursivo.
**Qué es 403:** el código HTTP "Forbidden", prohibido: el servidor entendió el pedido y se niega. `200 OK` es "bien". Lo ven con `curl -I` en los labs.
**Qué es `curl -I`:** `curl` (Días 1 y 4: pedir una página desde la terminal) con `-I`: pide solo el encabezado. La primera línea trae el código: `HTTP/1.1 200 OK` o `HTTP/1.1 403 Forbidden`. Sirve para no llenar la pantalla con HTML. Cuando queremos ver el contenido, `curl` sin `-I`.

---

## El problema que resuelve

**Qué decir (con tu sombrero):** "Si alguien explota Apache, su código corre como el usuario `apache`. Lo primero que va a intentar: leer `/etc/passwd` y `/etc/shadow`, dejar un archivo en una carpeta donde pueda escribir, conectarse a su servidor. Con solo `rwx`, varias de esas cosas funcionan. Con SELinux, `httpd_t` solo puede leer contenido web y escuchar en puertos web. Sigue comprometido, pero encerrado."
**Qué señalar en la tabla:** la fila de root: en DAC puede todo; en MAC está sujeto. "Por eso en el Lab 1.2 root recibió `Permission denied` al abrir el 82."
**Analogía:** DAC es el candado de tu casillero: vos decidís a quién le das la llave. MAC es el reglamento del edificio: aunque tengas la llave, el guardia no te deja subir a un piso que no corresponde a tu credencial. Las dos comprobaciones ocurren.

---

## Modos

`getenforce`, `sestatus`.
**Qué decir:** "`Enforcing` es producción. `Permissive` es la herramienta de diagnóstico: si en permissive funciona, es SELinux. Se vuelve a enforcing **antes** de corregir, no después. `Disabled` no se usa nunca."
**Qué señalar:** en `sestatus`, `Current mode` y `Mode from config file`: el primero es ahora (`setenforce`), el segundo es al arrancar (`/etc/selinux/config`). Cuando difieren, alguien hizo `setenforce 0` y se olvidó.

---

## Contextos: la etiqueta

Leer el diagrama de abajo hacia arriba y quedarse en el tipo. **Qué decir:** "cuatro campos, uno importa: el tipo, el que termina en `_t`."
Los tres comandos: `id -Z`, `ps -eZ | grep httpd`, `ls -Z /var/www/html/`.
**Qué señalar:** tu sesión es `unconfined_t` ("a mí no me limita"); Apache es `httpd_t`; el `index.html` es `httpd_sys_content_t`. "La política dice: `httpd_t` puede leer `httpd_sys_content_t`. Eso es todo lo que hace falta para que la web funcione."

---

## Tipos que hay que reconocer

No leer la tabla entera. Señalar cinco: `httpd_t` (el proceso), `httpd_sys_content_t` (lo que puede leer), `http_port_t` (donde puede escuchar), `admin_home_t` (lo de `/root`: la trampa del Lab 2.2), `default_t` (carpeta nueva sin regla: la trampa del Lab 3.2).

---

## De dónde sale la etiqueta

**La frase que hay que decir:** *"Un archivo nace con la etiqueta de la carpeta donde lo creás. Si lo copiás, el nuevo nace bien. Si lo movés, se lleva la etiqueta vieja puesta. Por eso 'lo copié y funciona, lo moví y da 403'."* Es exactamente lo que van a hacer en el Lab 2.2. Y la solución tiene un nombre: `restorecon`.
