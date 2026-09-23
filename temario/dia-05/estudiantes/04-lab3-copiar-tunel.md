# Lab — Copiar archivos y abrir un túnel

Mover archivos entre su computadora y la VM, mantener una carpeta sincronizada, y llegar a un servicio interno del servidor que el firewall no deja entrar.

---

## Parte 1 — Copiar un archivo en cada dirección

En **su computadora**:

```bash
echo "nota desde mi equipo" > nota.txt
scp nota.txt rhel01:empresa/documentos/
scp rhel01:/etc/hostname ./hostname-rhel01.txt
cat hostname-rhel01.txt
```
(En Windows, para crear el archivo: `"nota desde mi equipo" | Out-File -Encoding ascii nota.txt`)

Ahora ustedes: comprobar desde la VM que llegó, con `ls ~/empresa/documentos/`. Foto.

**Comprobar:** `hostname-rhel01.txt` contiene `rhel01.lab.local`, y en la VM está `nota.txt`.

La sintaxis es `scp origen destino`, y el lado remoto se escribe `host:ruta`. Una ruta remota **sin** `/` al principio es relativa al home.

---

## Parte 2 — Sincronizar una carpeta

Dentro de la **VM**:

```bash
rsync -avz --delete ~/empresa/ ~/empresa-copia/
```

**Comprobar:** lista todo lo que copió y termina con `sent ... received ... total size is ...`

Ahora cambiamos algo y repetimos:

```bash
touch ~/empresa/documentos/nuevo.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
```

**Comprobar:** la segunda vez **solo viaja `nuevo.txt`**. Eso es lo que hace útil a `rsync`: no vuelve a copiar lo que ya está igual.

Ahora ustedes: borren `~/empresa/documentos/nuevo.txt`, corran el `rsync` otra vez, y miren qué dice. Foto.

**Comprobar:** dice `deleting documentos/nuevo.txt` — el `--delete` borra en el destino lo que ya no está en el origen. Así se mantiene un espejo exacto.

---

## Parte 3 — Levantar un servicio interno

En la **VM**, dejar corriendo un servidor web de prueba:

```bash
cd ~/empresa
python3 -m http.server 8000
```

**Comprobar:** `Serving HTTP on 0.0.0.0 port 8000 ...` y la terminal queda ocupada. **Dejarla así.**

---

## Parte 4 — Comprobar que desde afuera no se llega

En **su computadora**, en otra ventana:

```bash
curl -m 3 http://192.168.56.10:8000/
```
(En Windows: `curl.exe`)

**Comprobar:** falla. `No route to host` o un tiempo de espera agotado. El firewall del servidor solo deja entrar SSH.

---

## Parte 5 — El túnel

Desde **su computadora**:

```bash
ssh -N -L 8081:localhost:8000 rhel01
```

La terminal queda ocupada, sin devolver el prompt. Es correcto.

En **otra ventana** (o en el navegador, `http://localhost:8081/`):

```bash
curl http://localhost:8081/
```

**Comprobar:** ahora sí responde: el listado de `~/empresa` (`documentos/`, `clientes/`, `backups/`, `logs/`). Y en la terminal de la VM aparece la petición registrada.

Cerrar el túnel con `Ctrl+C`, y el servidor de prueba en la VM también con `Ctrl+C`.

**Lo que acaba de pasar:** el puerto 8000 sigue cerrado desde afuera. El túnel entró por el 22 — que sí está permitido — y desde adentro de la VM habló con su propio `localhost:8000`, donde el firewall no interviene.

---

# Solución — todos los comandos

## Parte 1 — en su propia computadora
```bash
echo "nota desde mi equipo" > nota.txt
scp nota.txt rhel01:empresa/documentos/
scp rhel01:/etc/hostname ./hostname-rhel01.txt
cat hostname-rhel01.txt
```
En la VM, para comprobar: `ls ~/empresa/documentos/`

## Parte 2 — dentro de la VM
```bash
rsync -avz --delete ~/empresa/ ~/empresa-copia/
touch ~/empresa/documentos/nuevo.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
rm ~/empresa/documentos/nuevo.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
```

## Parte 3 — dentro de la VM
```bash
cd ~/empresa
python3 -m http.server 8000
```

## Parte 4 — en su propia computadora
```bash
curl -m 3 http://192.168.56.10:8000/
```
Tiene que fallar: el firewall solo deja entrar SSH.

## Parte 5 — en su propia computadora
```bash
ssh -N -L 8081:localhost:8000 rhel01
```
Y en otra ventana:
```bash
curl http://localhost:8081/
```
`Ctrl+C` para cerrar el túnel, y otro `Ctrl+C` en la VM para detener el servidor de prueba.
