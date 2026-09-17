# Lab — Calcular una red con `ipcalc`

Vamos a instalar las herramientas del día y a comprobar, a mano y con la herramienta, a qué red pertenece una IP.

---

## Parte 1 — Las herramientas del día

**¿Cómo instalo varios paquetes de una vez?**

```bash
sudo dnf install -y ipcalc bind-utils rsync tcpdump
```

Ahora ustedes. Foto.

**Comprobar:** termina en `Complete!`. Si `ipcalc` ya estaba, dice `Package ipcalc-... is already installed` y sigue con los demás.

---

## Parte 2 — La carpeta del Día 2 sigue ahí

```bash
ls ~/empresa ~/empresa/documentos
```

**Comprobar:**
```
/home/student/empresa:
backups  clientes  documentos  logs

/home/student/empresa/documentos:
informe1.txt  informe2.txt  informe3.txt  informe4.txt  informe5.txt  plan-2026.doc  presupuesto.csv
```
Puede haber algún archivo más; lo que importa son las cuatro carpetas y los informes. Si falta: `mkdir -p ~/empresa/documentos ~/empresa/clientes ~/empresa/backups ~/empresa/logs` y después `touch ~/empresa/documentos/informe1.txt`.

---

## Parte 3 — La red de la IP que vamos a usar hoy

**¿En qué red está `192.168.56.10` con máscara `/24`? ¿Cuál es su última dirección?**

```bash
ipcalc -bmn 192.168.56.10/24
```

Ahora ustedes: la IP de la VM en NAT, `10.0.2.15/24`. Foto.

**Comprobar** (el orden de las líneas puede variar):
```
NETMASK=255.255.255.0
BROADCAST=192.168.56.255
NETWORK=192.168.56.0
```
Para la de ustedes: `NETWORK=10.0.2.0` y `BROADCAST=10.0.2.255`.

---

## Parte 4 — Una máscara que no termina en 0

**¿`172.16.40.130/26` y `172.16.40.200/26` están en la misma red?**

```bash
ipcalc -bmn 172.16.40.130/26
```

Ahora ustedes: `172.16.40.200/26`. Foto.

**Comprobar:**
```
NETMASK=255.255.255.192
BROADCAST=172.16.40.191
NETWORK=172.16.40.128
```
Para `.200`: `NETWORK=172.16.40.192`. **No** están en la misma red: para hablarse necesitan un gateway. Con `/26` cada red tiene solo 62 direcciones para máquinas.
