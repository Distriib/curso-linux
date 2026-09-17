# Lab — Publicar Apache a través del firewall (comandos)

Todos tipean lo mismo; vos primero, ellos después, foto. Las pruebas "desde tu computadora" son en el navegador de cada uno, no en la VM: decirlo cada vez. Si tu VM es UTM, donde ellos ven `enp0s3 enp0s8` vos vas a ver otros nombres; avisalo en la Parte 3.

## Parte 1 — Instalar y arrancar
```bash
sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server dnf-automatic
sudo systemctl enable --now httpd
echo "<h1>Portal institucional - rhel01</h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```
`httpd` ya estaba desde el Día 4: dnf dice `already installed` y sigue con los otros tres. `setroubleshoot-server` arrastra unos 100 MB: dos o tres minutos. Mientras baja, seguir con la explicación.

## Parte 2 — Desde afuera
En el navegador de la computadora de cada uno: `http://localhost:8080` y `http://192.168.56.10`. Las dos fallan.
Qué decir: "Apache funciona, lo vimos con `curl` desde adentro. El firewall de la VM no deja entrar."

## Parte 3 — El estado del portero
```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

## Parte 4 — Los servicios que ya vienen definidos
```bash
sudo firewall-cmd --get-services | wc -w
sudo firewall-cmd --info-service=http
cat /usr/lib/firewalld/services/http.xml
ls /etc/firewalld/services/
```

## Parte 5 — Abrirlo mal: solo en runtime
```bash
sudo firewall-cmd --add-service=http
sudo firewall-cmd --list-services
```
Que prueben `http://localhost:8080` en el navegador: carga. Después:
```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```
Qué decir: "esto es lo que le pasa al que abre algo, funciona, y a los tres meses reinician el servidor y 'dejó de funcionar'."

## Parte 6 — Abrirlo bien
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --list-services
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --query-service=http
```
Parar en la segunda línea: "`http` todavía no está. `--permanent` escribió en disco y no tocó lo que corre. Ese es el otro error: 'lo puse permanente y no funciona'. Falta el `--reload`."

## Parte 7 — Un puerto suelto y --runtime-to-permanent
```bash
sudo firewall-cmd --add-port=7070/tcp
sudo firewall-cmd --list-ports
sudo firewall-cmd --permanent --list-ports
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --permanent --list-ports
sudo firewall-cmd --permanent --remove-port=7070/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```
El 7070 es un puerto cualquiera donde nadie escucha: es solo para ver el mecanismo.

## Parte 8 — Debajo de la alfombra
```bash
sudo nft list ruleset | grep "dport 80"
sudo nft list ruleset | wc -l
```
Si el `grep` no devuelve nada en alguna VM: `sudo nft list ruleset | grep dport` y mostrar lo que salga. El formato exacto cambia entre versiones; lo que se enseña es que cada `--add-service` termina siendo una línea del kernel.
