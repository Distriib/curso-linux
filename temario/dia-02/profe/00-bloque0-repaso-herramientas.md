# 0 — Repaso e instalaciones (guía del instructor)

## Conectarse

```bash
ssh -p 2222 student@localhost
hostname && cat /etc/redhat-release && uname -m
```

Salida esperada:
```
rhel01
Red Hat Enterprise Linux release 9.x (Plow)
x86_64
```

Qué decir: la versión menor (9.4, 9.5, 9.6...) depende de la ISO que cada uno
usó el Día 1; todos deben ver `9.x`, no tiene que ser idéntico. En tu VM la
arquitectura da `aarch64` en vez de `x86_64` — es esperable, no es un error.

## Confirmar el registro

```bash
sudo dnf repolist
```

**Qué es `dnf repolist`:** lista los repositorios (los "almacenes" de paquetes)
que el sistema tiene disponibles para instalar software. No instala nada, solo
muestra de dónde *podría* instalar.

Salida esperada — tienen que aparecer exactamente estos dos:
```
repo id                                   repo name
rhel-9-for-x86_64-appstream-rpms          Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms             Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
```

Si a alguien le sale vacío o dice `This system is not registered with an
entitlement server`: no está registrado. Solución en vivo:
```bash
sudo subscription-manager register
```
(usuario y contraseña de developers.redhat.com, del Día 1). Con Simple Content
Access no hace falta `attach` después de registrar.

## Instalar las herramientas de hoy

Uno por uno, explicando para qué sirve cada uno mientras se instala:

```bash
sudo dnf install -y tree
```
Para el **Lab 2.1**: ver `~/empresa` como diagrama de árbol en vez de listas
sueltas de `ls`.

```bash
sudo dnf install -y vim-enhanced
```
Para el **Lab 6.1**. RHEL siempre trae instalado `vim-minimal` (la versión
mínima, la que existe incluso en modo rescate) — funciona, pero sin colores
y sin algunos extras. Este paquete (`vim-enhanced`) le agrega dos cosas:

- **Colores** (resaltado de sintaxis): pinta el texto según lo que sea
  (comentarios, palabras clave, etc.), se lee más fácil.
- **El comando `vimtutor`**: abre un tutorial interactivo *dentro de la
  terminal* — un archivo de práctica con instrucciones que se van resolviendo
  haciendo, no leyendo (moverse, borrar, guardar…), lección por lección. Es
  justo la tarea que les vas a mandar hoy ("`vimtutor`, lecciones 1 a 4").
  Sin `vim-enhanced` ese comando ni existe.

Si ya estaba instalado desde el Día 1, `dnf` avisa `Package ... is already
installed` y sigue sin problema.

```bash
sudo dnf install -y bash-completion
```
Para que **Tab** complete subcomandos (`systemctl sta<Tab>` → `start`), no
solo nombres de archivo. Se usa desde el Lab 1.1.

```bash
sudo dnf install -y zip
sudo dnf install -y unzip
```
Para el **Lab 5.1**, comparar el formato `.zip` (el que entienden los usuarios
de Windows) contra `tar`.

```bash
sudo dnf install -y bzip2
```
Para el **Lab 5.1**: uno de los tres compresores que se comparan (`gzip`,
`bzip2`, `xz`).

```bash
sudo dnf install -y nano
```
Para el **Lab 6.1**, como alternativa más simple a vim.

Si algún paquete ya estaba instalado (`vim-enhanced`, `bash-completion` y
`nano` suelen venir de fábrica en Server), `dnf` responde `Package ... is
already installed` y sigue sin error — no hay que hacer nada distinto.

**Todo junto, la forma rápida** (mostrarla al final, como la manera en que
realmente se hace en el trabajo diario — un solo `dnf` resuelve las
dependencias de los siete paquetes de una sola vez, en vez de siete
transacciones separadas):

```bash
sudo dnf install -y tree vim-enhanced bash-completion zip unzip bzip2 nano
```

`mlocate` (para `locate`, Bloque 4) se instala más tarde, en su propio bloque,
no acá.
