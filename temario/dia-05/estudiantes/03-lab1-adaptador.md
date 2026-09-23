# Lab — Agregar una segunda tarjeta de red

Hoy la VM solo tiene una tarjeta, la del NAT, y por eso hay que entrar con `-p 2222`. Vamos a agregarle una segunda, en una red privada entre su PC y la VM, para entrar **directo por IP**.

---

## Parte 1 — Apagar la VM

```bash
sudo poweroff
```

**Comprobar:** la sesión SSH se cierra sola. Es lo esperado.

---

## Parte 2 — Agregar la tarjeta en el hipervisor

**VirtualBox:**

1. Menú *File > Tools > Network Manager*, pestaña *Host-only Networks*. Tiene que existir una red con IPv4 `192.168.56.1/24`. Si no existe, botón *Create*.
2. Seleccionar la VM `rhel01` > *Settings > Network > Adapter 2*.
3. Marcar *Enable Network Adapter*.
4. *Attached to:* **Host-only Adapter**. *Name:* la red del paso 1.
5. *OK*.

**UTM:**

1. Con la VM apagada, abrir su configuración.
2. En la lista de dispositivos, *New... > Network*.
3. *Network Mode:* **Host Only**. *Emulated Network Card:* `virtio-net-pci`.
4. *Save*.
5. El primer adaptador **no se toca**: es el que tiene el reenvío de puertos.

**Comprobar:** en la configuración de la VM ahora hay dos adaptadores de red.

---

## Parte 3 — Encender y buscar la tarjeta nueva

Encender la VM, esperar, y volver a entrar:

```bash
ssh -p 2222 student@localhost
nmcli device status
ip -br a
```

**Comprobar:** aparece una tarjeta más.

```
DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
enp0s8  ethernet  disconnected            --
lo      loopback  connected (externally)  lo
```

La tarjeta nueva puede aparecer `disconnected` y sin perfil, o `connected` con un perfil llamado `Wired connection 1` que NetworkManager crea solo. **Las dos situaciones son normales**; en el próximo lab le ponemos el perfil nuestro.

**Anoten el nombre de la tarjeta nueva.** En VirtualBox suele ser `enp0s8`; en UTM, `enp0s2`. Se usa en todo el resto del día.

---

## Parte 4 — Que la ruta por defecto siga siendo la del NAT

```bash
ip route
```

**Comprobar:** la línea `default via 10.0.2.2` sigue ahí, saliendo por `enp0s3`. La tarjeta nueva **no** debe tener ruta por defecto: esa red no lleva a internet.

---

# Solución — todos los comandos

## Parte 1
```bash
sudo poweroff
```

## Parte 2
Se hace en la ventana del hipervisor, no en la terminal.

En **VirtualBox**, la alternativa por línea de comandos desde el equipo propio, con la VM apagada:
```bash
VBoxManage modifyvm "rhel01" --nic2 hostonly --host-only-adapter2 "VirtualBox Host-Only Ethernet Adapter"
```

## Parte 3
```bash
ssh -p 2222 student@localhost
nmcli device status
ip -br a
```

## Parte 4
```bash
ip route
```
La línea `default via 10.0.2.2` tiene que seguir saliendo por `enp0s3`.
