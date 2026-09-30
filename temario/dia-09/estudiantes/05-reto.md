# Reto — Ticket #0931

**Hacelo sin mirar los labs.** Todo lo que hace falta está en los labs de hoy; si te trabás en un punto, la solución de ese paso está al final del lab correspondiente.

> **Portal institucional en contenedor**
> Solicitante: Dirección de Informática. Servidor: `rhel01`.

## Lo que piden

1. Un contenedor llamado `portal`, con la imagen `registry.access.redhat.com/ubi9/httpd-24`, que sirva la carpeta `~/portal` del usuario `student`. Adentro tiene que haber un `index.html` con el texto `Portal PGN`. Responde en el puerto **8085** del servidor.

2. El contenedor se maneja como servicio de `student` con **Quadlet** (`portal.service`), y **arranca solo al reiniciar la VM, sin que nadie inicie sesión**.

3. El portal se abre desde el navegador de tu computadora: `http://192.168.56.10:8085/`.

4. La carpeta `~/portal` se exporta por **NFS** a la red host-only (`192.168.56.0/24`) en modo **solo lectura**.

5. Esa carpeta se monta a demanda con **autofs** en `/remoto/portal`.

## Verificación

Reiniciar la VM con `sudo reboot`. **Antes de entrar por SSH**, abrir `http://192.168.56.10:8085/` en el navegador: si carga sin que nadie haya iniciado sesión, el *linger* y el servicio están bien. Después entrar y correr:

```bash
systemctl --user is-active portal.service
podman ps
curl http://localhost:8085/
loginctl show-user student | grep Linger
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
showmount -e localhost
cat /remoto/portal/index.html
mount | grep portal
```

Está bien si: la página del navegador muestra `Portal PGN`; el servicio está `active`; `podman ps` lista `portal` con `0.0.0.0:8085->8080/tcp`; el `curl` devuelve `Portal PGN`; `Linger=yes`; `8085/tcp` aparece en las **dos** zonas; `showmount` lista `/home/student/portal 192.168.56.0/24`; el `cat` muestra `Portal PGN`; y `mount` muestra `/home/student/portal on /remoto/portal type nfs4`.
