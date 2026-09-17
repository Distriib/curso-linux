# Lab — La red de administración en su zona, y un Apache que no arranca (comandos)

Vos primero, ellos después, foto. Si tu VM es UTM, tu red host-only no es `192.168.56.0/24`: usá la tuya en el `--add-source` y avisá que ellos (VirtualBox) usan la que dice el archivo. Hoy nadie toca `ssh` en ninguna zona: nadie se queda afuera.

## Parte 1 — Conocer las zonas
```bash
sudo firewall-cmd --get-zones
sudo firewall-cmd --info-zone=drop | head -3
sudo firewall-cmd --info-zone=trusted | head -3
sudo firewall-cmd --info-zone=public | head -3
```

## Parte 2 — La red host-only es de administración
```bash
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24
sudo firewall-cmd --reload
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=internal --list-all
```
Qué decir: "ahora hay dos zonas en uso: `internal` por origen, `public` por interfaz. Fijate que `internal` no tiene `http`."

## Parte 3 — Probar las dos rutas
En el navegador de cada uno: `http://localhost:8080` (carga) y `http://192.168.56.10` (no carga).
Dar el minuto de la pregunta antes de responder. La respuesta: "lo que viene de `192.168.56.x` ahora cae en `internal` por el origen, y en `internal` no está `http`. El origen manda sobre la interfaz." Es **el** momento del bloque: que lo digan ellos.

## Parte 4 — Corregir
```bash
sudo firewall-cmd --permanent --zone=internal --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --zone=internal --list-services
```
Que prueben de nuevo las dos rutas. Qué decir: "desde hoy y hasta el final del curso: todo lo que se publica se abre en `public` y en `internal`."

## Parte 5 — El encargo: Apache también en el puerto 82
```bash
echo "Listen 82" | sudo tee /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
systemctl status httpd --no-pager -l
```
`Listen` es la directiva de Apache que dice en qué puerto escuchar. Su configuración principal, `/etc/httpd/conf/httpd.conf`, trae `Listen 80` y al final incluye todos los archivos `/etc/httpd/conf.d/*.conf`: un archivo nuevo ahí con `Listen 82` hace que Apache intente escuchar en el 80 **y** en el 82. Con un solo puerto que no pueda abrir, Apache no arranca. El `-l` de `systemctl status` es "líneas completas, sin recortar".
Qué decir: "`Permission denied`. Apache arranca como root, y root con permisos `rwx` puede abrir cualquier puerto. Hay alguien más diciendo que no. Lo dejamos roto: es el punto de partida del siguiente bloque."
No arreglar nada acá aunque alguien sepa cómo (`setenforce 0` en especial): decir "lo vamos a arreglar bien, no apagando cosas".
