# Lab — IP fija con el perfil `lab`

Dejar la tarjeta nueva con la dirección `192.168.56.10`, fija, que sobreviva a los reinicios, y entrar desde su PC directo a esa IP.

> Todo este lab se hace desde la sesión `ssh -p 2222 student@localhost`.
> En UTM, usar una IP del rango que indique el instructor.

---

## Parte 1 — Mirar cómo es un perfil por dentro

```bash
sudo ls -l /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/enp0s3.nmconnection
```

**Comprobar:** un archivo `.nmconnection`, de `root`, con permisos `-rw-------`, y adentro secciones como `[ipv4]` con `method=auto`.

---

## Parte 2 — Crear el perfil

**¿Cómo creo un perfil con IP fija para la tarjeta nueva?**

Una sola línea (cambiar `enp0s8` por el nombre de **su** tarjeta):

```bash
sudo nmcli connection add type ethernet con-name lab ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.56.10/24
```

**Comprobar:** `Connection 'lab' (...) successfully added.`

**Sin gateway, a propósito:** esta red no sale a internet. Si le pusieran gateway, la VM intentaría salir por donde no hay nada.

---

## Parte 3 — Aplicarlo y verificar

```bash
sudo nmcli connection up lab
ip -br a
ip route
nmcli device status
```

**Comprobar:**
```
enp0s8    UP    192.168.56.10/24 fe80::.../64
default via 10.0.2.2 dev enp0s3 ...
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.10 metric 101
enp0s8  ethernet  connected  lab
```
La tarjeta se llama `enp0s8` y el perfil se llama `lab`: ahí se ve la diferencia entre **tarjeta** y **perfil**.

---

## Parte 4 — Sacar el perfil automático

Si NetworkManager había creado uno solo, sobra:

```bash
sudo nmcli connection delete "Wired connection 1"
nmcli connection show
sudo cat /etc/NetworkManager/system-connections/lab.nmconnection
```

**Comprobar:** en la lista quedan `enp0s3`, `lab` y `lo`. En el archivo nuevo: `method=manual` y `address1=192.168.56.10/24`.

Si dice `Error: unknown connection 'Wired connection 1'`, es que no existía. No pasa nada, seguir.

---

## Parte 5 — Entrar desde su PC, sin `-p 2222`

En **su propia computadora** (PowerShell o Terminal), no en la VM:

```bash
ping -c 2 192.168.56.10
ssh student@192.168.56.10
hostname
exit
```
(En Windows: `ping -n 2 192.168.56.10`)

**Comprobar:** pregunta por la huella del servidor la primera vez (responder `yes`), pide la contraseña, y entra. `hostname` dice `rhel01`.

Esta es la primera vez que entran a la VM **como a un servidor de verdad**: por su IP y su puerto 22, sin la traducción del hipervisor.

---

## Parte 6 — Comprobar que la configuración es persistente

De vuelta en la sesión `ssh -p 2222`:

```bash
sudo nmcli connection down lab
sudo nmcli connection up lab
ip -br a show enp0s8
```

**Comprobar:** después de bajarla y levantarla, la IP sigue siendo `192.168.56.10/24`. Está guardada en el archivo, no puesta a mano.

---

# Solución — todos los comandos

```bash
# Parte 1 — mirar un perfil existente
sudo ls -l /etc/NetworkManager/system-connections/
sudo cat /etc/NetworkManager/system-connections/enp0s3.nmconnection

# Parte 2 — crear el perfil (una sola línea)
sudo nmcli connection add type ethernet con-name lab ifname enp0s8 ipv4.method manual ipv4.addresses 192.168.56.10/24

# Parte 3 — aplicar y verificar
sudo nmcli connection up lab
ip -br a
ip route
nmcli device status

# Parte 4 — quitar el perfil automático
sudo nmcli connection delete "Wired connection 1"
nmcli connection show
sudo cat /etc/NetworkManager/system-connections/lab.nmconnection

# Parte 6 — comprobar que es persistente
sudo nmcli connection down lab
sudo nmcli connection up lab
ip -br a show enp0s8
```

## Parte 5 — desde su propia computadora
```bash
ping -c 2 192.168.56.10
ssh student@192.168.56.10
hostname
exit
```
