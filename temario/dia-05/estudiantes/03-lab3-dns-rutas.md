# Lab — Modificar el perfil: DNS, dos IPs, ruta y nombre

Practicar `nmcli connection modify` con lo que pide el examen, y comprobar cada cambio en el sistema.

> Todo desde la sesión `ssh -p 2222 student@localhost`.
> **Recordá:** `modify` escribe el archivo, `up` lo aplica. Sin `up` no pasa nada.

---

## Parte 1 — DNS en el perfil

**¿Cómo le pongo servidores DNS a este perfil?**

```bash
sudo nmcli connection modify lab ipv4.dns 1.1.1.1 +ipv4.dns 8.8.8.8 ipv4.dns-search lab.local
cat /etc/resolv.conf
```

**Comprobar:** `/etc/resolv.conf` **todavía no cambió**. Eso es lo que hay que ver.

Ahora sí:

```bash
sudo nmcli connection up lab
cat /etc/resolv.conf
```

**Comprobar:** ahora aparecen `search lab.local` y los `nameserver` nuevos. El archivo lo reescribió NetworkManager solo — por eso no se edita a mano.

---

## Parte 2 — Dos IPs en la misma tarjeta

**¿Se puede tener dos direcciones en una sola tarjeta?** Sí; se usa para publicar dos servicios en IPs distintas.

```bash
sudo nmcli connection modify lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"
sudo nmcli connection up lab
ip -br a show enp0s8
```

Ahora ustedes: desde su PC, `ping 192.168.56.11`. Foto.

**Comprobar:**
```
enp0s8    UP    192.168.56.10/24 192.168.56.11/24 fe80::.../64
```

---

## Parte 3 — Una ruta hacia otra red

**¿Cómo le digo "para llegar a la red X, pasá por Y"?**

```bash
sudo nmcli connection modify lab +ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route
```

**Comprobar:** aparece una línea nueva:
```
10.10.0.0/24 via 192.168.56.1 dev enp0s8 proto static metric 101
```
`proto static` = la puso una persona, y sobrevive al reinicio.

Ahora hay que quitarla, porque apunta a un equipo que no existe y molestaría en el reto:

```bash
sudo nmcli connection modify lab -ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route
```

**Comprobar:** la línea de `10.10.0.0/24` ya no está.

---

## Parte 4 — Nombre completo del servidor y nombres locales

```bash
sudo hostnamectl set-hostname rhel01.lab.local
hostnamectl --static
hostname
```

**Comprobar:** dice `rhel01.lab.local`. **El prompt va a seguir diciendo `rhel01`** — solo muestra la parte anterior al primer punto. No está mal.

Ahora los nombres locales, que se van a usar los próximos días:

```bash
echo "192.168.56.10 rhel01.lab.local rhel01" | sudo tee -a /etc/hosts
echo "192.168.56.10 servidor-nfs servidor-smb" | sudo tee -a /etc/hosts
getent hosts servidor-nfs
ping -c 1 rhel01.lab.local
```

Ahora ustedes: `getent hosts servidor-smb`. Foto.

**Comprobar:** los dos nombres responden con `192.168.56.10`. Son alias de la propia VM: así los labs de los próximos días pueden usar nombres en vez de IPs, como en un servidor real.

---

## Parte 5 — Ver todo el perfil de una vez

```bash
nmcli connection show lab | grep ipv4
```

**Comprobar:** en la lista tienen que estar `ipv4.method: manual`, las dos direcciones, y los DNS que pusieron.

---

# Solución — todos los comandos

```bash
# Parte 1 — DNS
sudo nmcli connection modify lab ipv4.dns 1.1.1.1 +ipv4.dns 8.8.8.8 ipv4.dns-search lab.local
cat /etc/resolv.conf
sudo nmcli connection up lab
cat /etc/resolv.conf

# Parte 2 — dos IPs
sudo nmcli connection modify lab ipv4.addresses "192.168.56.10/24,192.168.56.11/24"
sudo nmcli connection up lab
ip -br a show enp0s8

# Parte 3 — una ruta, y quitarla
sudo nmcli connection modify lab +ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route
sudo nmcli connection modify lab -ipv4.routes "10.10.0.0/24 192.168.56.1"
sudo nmcli connection up lab
ip route

# Parte 4 — nombre del servidor y nombres locales
sudo hostnamectl set-hostname rhel01.lab.local
hostnamectl --static
hostname
echo "192.168.56.10 rhel01.lab.local rhel01" | sudo tee -a /etc/hosts
echo "192.168.56.10 servidor-nfs servidor-smb" | sudo tee -a /etc/hosts
getent hosts servidor-nfs
getent hosts servidor-smb
ping -c 1 rhel01.lab.local

# Parte 5 — ver el perfil completo
nmcli connection show lab | grep ipv4
```
