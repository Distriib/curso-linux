# Lab — Segunda tarjeta de red (guía)

**Qué es la red host-only:** una red privada entre el equipo del participante y su VM. No sale a internet. Sirve para entrar a la VM por su IP, como a un servidor de verdad, sin el reenvío de puertos.
**Por qué hace falta apagar la VM:** VirtualBox no permite habilitar un adaptador nuevo con la VM encendida.

## Parte 1
```bash
sudo poweroff
```

## Parte 2
Por la interfaz del hipervisor. En **VirtualBox**, la alternativa por línea de comandos en el host, con la VM apagada:
```bash
VBoxManage modifyvm "rhel01" --nic2 hostonly --host-only-adapter2 "VirtualBox Host-Only Ethernet Adapter"
```
(En Mac el nombre del adaptador se ve con `VBoxManage list hostonlyifs`.)

En **UTM** la red host-only la crea macOS y **el rango lo decide macOS**, no vos. Para saber cuál te tocó, en la terminal del Mac con la VM ya encendida:
```bash
ifconfig | grep -A 4 bridge100
```
La línea `inet 192.168.64.1` te dice la red. Donde el material diga `192.168.56.10`, usá una IP de **tu** rango (por ejemplo `192.168.64.10`).

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

---

## Qué señalar

- **Antes de empezar:** avisar que se apaga la VM y se corta el SSH. Si no, alguien cree que se rompió.
- **Parte 3:** los dos resultados posibles (tarjeta `disconnected` sin perfil, o `connected` con `Wired connection 1`) dependen de si la instalación trae el paquete `NetworkManager-config-server`. **Los dos son correctos** y el lab siguiente los deja iguales. Decirlo antes de que alguien pregunte por qué su pantalla no coincide con la del compañero.
- **Parte 3:** insistir en que anoten el nombre de la tarjeta nueva. Es el error que arrastra todo el bloque.
- **Parte 4:** la ruta por defecto tiene que seguir siendo la del NAT. Frase: *"la tarjeta nueva es para entrar, no para salir. Internet sigue saliendo por la otra"*.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| No aparece la segunda tarjeta | No marcaron *Enable Network Adapter*, o lo configuraron con la VM encendida |
| En VirtualBox no existe la red host-only | Hay que crearla en *File > Tools > Network Manager > Create* |
| La tarjeta nueva tomó la ruta por defecto | Le llegó un gateway por DHCP; se arregla solo en el próximo lab, al ponerle el perfil `lab` sin gateway |
| La VM no vuelve por SSH | Tarda; esperar un minuto más. Si no, mirar la ventana del hipervisor: puede estar pidiendo login |
| En UTM la red no es `192.168.56.x` | Correcto: macOS elige el rango. Usar el propio |
