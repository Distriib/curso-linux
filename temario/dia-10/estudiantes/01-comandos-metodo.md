# 1 — Método de diagnóstico

## Un ticket es un síntoma, no una causa

El usuario dice "la web da error". Nunca dice "el contexto SELinux de `index.html` está mal". Convertir lo uno en lo otro es el trabajo, y se hace con un procedimiento, no con intuición.

| Paso | Qué se hace | Con qué |
|---|---|---|
| 1 Síntoma | Qué reporta el usuario, desde cuándo, para quién, **qué cambió** | `last`, `sudo dnf history`, `journalctl --since` |
| 2 Qué debería pasar | Escribir la cadena completa: "el navegador llega al puerto 80, el firewall deja pasar, httpd escucha, lee el archivo, SELinux lo permite" | papel |
| 3 Hipótesis | Una por capa, de la más barata de comprobar a la más cara | la tabla de abajo |
| 4 Prueba | **Un** comando que confirme o descarte. Si no confirma, siguiente hipótesis | la tabla de abajo |
| 5 Corrección | Mínima y **persistente**: `enable --now`, `--permanent`, `restorecon`, `nmcli` | |
| 6 Verificación | Desde donde el usuario lo ve (el navegador de su computadora), no `curl localhost`. Y tras un reinicio si la corrección lo justifica | |
| 7 Documentación | Cuatro líneas: qué estaba mal · cómo lo encontré · cómo lo corregí · cómo lo verifiqué | el ticket |

## Las capas, de abajo hacia arriba

Se avanza a la siguiente solo cuando la actual está sana. Si el edificio no tiene luz, no se revisa la impresora.

```
1 Arranque          ¿la VM enciende, llega a GRUB, systemd terminó de arrancar?
2 Kernel            ¿el kernel ve los discos y las tarjetas de red?
3 Servicio          ¿la unidad está activa, habilitada, sin fallos?
4 Red / firewall    ¿hay IP, el puerto escucha, el firewall deja pasar?
5 Permisos          ¿dueño, grupo, rwx, setgid, ACL?
6 SELinux           ¿modo, contexto del archivo, puerto?
7 Almacenamiento    ¿hay espacio, está montado, el fstab está bien?
8 Aplicación        ¿qué dice su propio log?
```

## El primer comando de cada capa

| Capa | Pregunta | Primer comando | Después |
|---|---|---|---|
| Arranque | ¿Qué falló al arrancar? ¿Cuánto tardó? | `systemctl --failed` | `journalctl -b -p err`, `systemd-analyze blame` |
| Kernel | ¿El kernel ve el disco y la red? ¿Se quedó sin memoria? | `sudo dmesg -T \| tail` | `sudo dmesg -T \| grep -i oom`, `lsblk`, `ip link` |
| Servicio | ¿Está activo y habilitado? ¿Por qué se cayó? | `systemctl status SERVICIO` | `journalctl -u SERVICIO --since "1 hour ago"`, `journalctl -xe` |
| Red / firewall | ¿Escucha? ¿Qué proceso? ¿El firewall deja pasar? | `sudo ss -tlnp` | `ip -br a`, `ip route`, `sudo firewall-cmd --list-all`, `sudo firewall-cmd --get-active-zones` |
| Permisos | ¿Quién es el usuario y qué puede tocar? | `ls -ld RUTA` | `id USUARIO`, `getfacl`, `sudo passwd -S USUARIO`, `sudo -l -U USUARIO` |
| SELinux | ¿Está bloqueando? | `sudo ausearch -m AVC -ts recent` | `getenforce`, `ls -Z`, `sudo semanage port -l \| grep NOMBRE` |
| Almacenamiento | ¿Hay espacio? ¿Está montado? | `df -h` | `df -i`, `sudo du -sh /ruta`, `sudo lsof +L1`, `findmnt --verify` |
| Rendimiento | ¿CPU, memoria, carga? | `uptime` | `free -m`, `top`, `vmstat 1 3` |
| Sesiones | ¿Quién entró, quién está, quién falló? | `who` | `last`, `sudo lastb`, `sudo grep Failed /var/log/secure` |
| Aplicación | ¿Qué dice ella misma? | su log en `/var/log/NOMBRE/` | `sudo apachectl configtest`, `sudo sshd -t`, `sudo rsyslogd -N1` |

```bash
systemctl --failed
sudo ss -tlnp
df -h
```

## Tres reglas para hoy

1. **Reproducir antes de arreglar.** Si no se puede ver el error, no se puede saber si se corrigió.
2. **Un cambio a la vez, y anotarlo.** Si se tocan tres cosas y funciona, no se sabe cuál era.
3. **`setenforce 0`, `systemctl stop firewalld` y `chmod 777` no son diagnósticos.** Esconden la causa. En el examen, descalifican.
