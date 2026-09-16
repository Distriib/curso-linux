# Reto — Ticket #2026-0312

**Solos, sin ayuda.** Si no se termina en clase, queda de tarea.

> **Alta del proyecto "Expediente Digital"**
> Solicitante: Coordinación de Proyectos. Servidor: `rhel01`.

## Lo que piden

1. Crear el grupo `expediente` (GID 3010) y los usuarios `maria` (UID 2010) y `jorge` (UID 2011), ambos con `expediente` como grupo suplementario, contraseña inicial `Pgn.2026`, contraseña que caduca cada **60 días** con aviso **10 días** antes, y **cuenta** que expira el **2026-12-31** (contrato por proyecto).

2. Crear la carpeta compartida `/srv/expediente`: dueño `root`, grupo `expediente`. Solo los miembros del grupo entran, leen y escriben. Todo archivo o subcarpeta nuevo pertenece al grupo `expediente` automáticamente. Nadie más entra.

3. Crear el usuario `auditor_ext` (UID 2012, contraseña `Pgn.2026`), que **no** es miembro de `expediente`, con permiso de **solo lectura** sobre `/srv/expediente` y sobre todo lo que se cree adentro en el futuro.

4. Los miembros de `expediente` pueden ejecutar como root **únicamente** `systemctl status chronyd` y `systemctl restart chronyd`. La regla va en su propio archivo bajo `/etc/sudoers.d/`. `laura` sigue sin ningún privilegio.

## Verificación

Pegar en el chat la salida completa de:

```bash
id maria; id jorge; id auditor_ext
sudo chage -l maria
ls -ld /srv/expediente
sudo -u maria bash -c 'echo caso > /srv/expediente/caso-001.txt'; sudo ls -l /srv/expediente
sudo -u auditor_ext cat /srv/expediente/caso-001.txt
sudo -u auditor_ext touch /srv/expediente/x.txt
sudo -u laura ls /srv/expediente
sudo -l -U jorge | tail -1
sudo -l -U laura
sudo visudo -c
```

Está bien si: `maria` y `jorge` tienen `3010(expediente)`; `auditor_ext` no; la cuenta expira `Dec 31, 2026`, máximo 60, aviso 10; la carpeta es `drwxrws---+ root expediente`; `caso-001.txt` tiene grupo `expediente`; `auditor_ext` lee `caso` pero recibe `Permission denied` al crear; `laura` recibe `Permission denied`; `jorge` puede los dos `systemctl`; `laura` "is not allowed to run sudo"; `parsed OK`.
