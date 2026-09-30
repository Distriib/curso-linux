# Reto — Ticket PGN-2026-0812

**Hacelo sin mirar los labs.** Todo lo que hace falta lo viste hoy; si te trabás, la solución de cada paso está en el lab correspondiente.

> **Portal institucional: nuevo puerto y nueva carpeta**
> Prioridad ALTA. Solicitante: Dirección de Informática. Servidor: `rhel01`.
> "El proveedor del portal ahora publica en el puerto **8082** y el contenido va en **/sitio**. Necesitamos que cargue hoy desde las computadoras de la oficina en `http://192.168.56.10:8082`."

**Prohibido:** `setenforce 0`, `SELINUX=permissive` o `disabled`, `systemctl stop firewalld`, zona `trusted`, `chcon` como solución final.

## Lo que piden

1. Apache escucha en el puerto **8082** (y ya no en el 82). SELinux lo tiene que permitir de forma persistente. Ojo: el 8082 ya tiene dueño.

2. El contenido se sirve desde **`/sitio`**, con un `index.html` que diga `Portal institucional - rhel01 - puerto 8082`. La etiqueta tiene que sobrevivir a un `restorecon`.

3. El **8082** abierto en el firewall de forma permanente en las dos zonas (`public` e `internal`). El **82** cerrado en las dos.

4. SELinux sigue en `Enforcing`, firewalld corriendo, y ningún servicio en permissive.

5. La página carga desde el navegador de tu computadora en `http://192.168.56.10:8082`.

## Verificación

Cuando creas que está listo, correr esto y revisar la salida:

```bash
getenforce
systemctl is-active httpd firewalld
sudo semanage port -l -C
sudo semanage fcontext -l -C
ls -Zd /sitio
curl http://localhost:8082
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
sudo semanage permissive -l
```

Está bien si: `Enforcing`; `active` y `active`; en los puertos locales aparece `http_port_t tcp 8082`; en los contextos locales aparece `/sitio(/.*)?` con `httpd_sys_content_t`; `/sitio` es `httpd_sys_content_t`; `curl` muestra el `<h1>`; las dos zonas dicen `8082/tcp` y ninguna `82/tcp`; en permissive no aparece `httpd_t` bajo `Customized`.
