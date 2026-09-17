# Lab 3.1 — Apache en el puerto 82

Vamos a resolver el fallo del Bloque 1 como corresponde: registrando el 82 como puerto web en SELinux, y abriéndolo en el firewall. El ejercicio clásico del examen.

---

## Parte 1 — Preparar el traductor y ver qué puertos conoce SELinux

**¿En qué puertos deja SELinux escuchar a Apache?**

```bash
sudo service auditd restart
sudo semanage port -l | grep -w http_port_t
sudo semanage port -l | grep 8080
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
```
El 82 no está en ningún lado (los puertos menores a 1024 sin dueño son `reserved_port_t`). El 8080 ya tiene dueño, `http_cache_port_t`, y a Apache la política **sí** lo deja usarlo. Por eso muchos tutoriales con 8080 "funcionan solos". Usamos el 82 porque falla, que es lo que queremos aprender a arreglar.

---

## Parte 2 — Reproducir el fallo y leer el AVC

**¿Qué dice exactamente la denegación?**

```bash
echo "Listen 82" | sudo tee /etc/httpd/conf.d/puerto.conf
sudo systemctl restart httpd
sudo ausearch -m AVC -ts recent | tail -1
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
Job for httpd.service failed ...
type=AVC msg=audit(...): avc:  denied  { name_bind } for  pid=... comm="httpd" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
```
Con la tabla de decisión: `tclass=tcp_socket`, `{ name_bind }`, `src=82` → puerto sin etiqueta → `semanage port -a`.

---

## Parte 3 — Registrar el puerto

**¿Cómo le digo a SELinux que el 82 es un puerto web?**

```bash
sudo semanage port -a -t http_port_t -p tcp 82
sudo semanage port -l | grep -w http_port_t
sudo systemctl restart httpd
curl http://localhost:82
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
http_port_t                    tcp      82, 80, 81, 443, 488, 8008, 8009, 8443, 9000
<h1>Portal institucional - version 2</h1>
```

---

## Parte 4 — Lo tuyo, y un puerto que ya tiene dueño

**¿Qué pasa si intento agregar un puerto que ya está definido?**

```bash
sudo semanage port -l -C
sudo semanage port -a -t http_port_t -p tcp 8080
```

Ahora ustedes: lo mismo. Foto.

**Comprobar:**
```
SELinux Port Type              Proto    Port Number

http_port_t                    tcp      82
ValueError: Port tcp/8080 already defined
```
Para reasignar un puerto que ya tiene dueño se usa `-m` (modificar) en vez de `-a`. Acordate de esto para el reto.

---

## Parte 5 — Abrirlo en el firewall, en las dos zonas

**SELinux ya deja; ¿y el firewall?**

```bash
sudo firewall-cmd --permanent --add-port=82/tcp
sudo firewall-cmd --permanent --zone=internal --add-port=82/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
sudo firewall-cmd --zone=internal --list-ports
```

Ahora ustedes: lo mismo, y en el navegador de su computadora `http://192.168.56.10:82`. Foto.

**Comprobar:**
```
success
success
success
82/tcp
82/tcp
```
Y la página carga en el 82 desde tu computadora.
