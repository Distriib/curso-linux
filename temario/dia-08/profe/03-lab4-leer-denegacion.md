# Lab — Leer una denegación de punta a punta (comandos)

Vos primero, ellos después, foto. Necesita el `service auditd restart` del Lab 3.1, Parte 1. Si en la Parte 3 el journal sale vacío: esperar 10 segundos y repetir; si sigue vacío, `sudo service auditd restart` y volver a provocar el 403 (Parte 1).

## Parte 1 — Provocar el problema
```bash
echo "parametros internos del portal" | sudo tee /root/config.txt
sudo mv /root/config.txt /web/
curl -I http://localhost:82/config.txt
```

## Parte 2 — El AVC crudo
```bash
sudo ausearch -m AVC -ts recent | tail -1
```
Leerlo en voz alta con la tabla de campos, campo por campo, y que alguien diga la fila de la tabla de decisión (fila 1: archivo movido → `restorecon`).

## Parte 3 — La versión traducida
```bash
sudo journalctl -t setroubleshoot --since "5 min ago" --no-pager
```
Copiar el código del final de la línea (después de `sealert -l`), y:
```bash
sudo sealert -l CODIGO
```
Cada VM tiene su propio código: nadie copia el tuyo. Qué decir: "las sugerencias vienen ordenadas por confianza. La primera, 99.5, dice `restorecon`. La última, `catchall`, con 1.5, propone fabricar una regla nueva con `audit2allow`: aparece **siempre** y casi nunca es la correcta. Le daría a Apache permiso para leer archivos de /root para siempre."
Si a alguien el journal le sigue vacío: `sudo sealert -a /var/log/audit/audit.log | tail -40` analiza el log completo sin depender del servicio.

## Parte 4 — Corregir y verificar
```bash
sudo restorecon -v /web/config.txt
curl http://localhost:82/config.txt
```

## Parte 5 — Permissive para un solo servicio
```bash
sudo semanage permissive -a httpd_t
sudo semanage permissive -l
getenforce
sudo semanage permissive -d httpd_t
```
El `-a` tarda unos segundos (compila un módulo). Qué decir: "el sistema sigue en `Enforcing`. Solo `httpd_t` quedó sin bloqueo, y seguía registrando. Esto en vez de `setenforce 0` cuando no podés parar. Y se quita."
