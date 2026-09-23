# 3 — NetworkManager (guía del instructor)

Diez minutos. Es el bloque más largo del día y el que más se atrasa: leé rápido y dejá que los labs expliquen.

**Qué es NetworkManager:** el programa que maneja la red en RHEL 9. Guarda la configuración en **perfiles** y se los aplica a las tarjetas. Nada de red se configura por fuera de él.
**Qué es un perfil (connection):** un archivo con la configuración: qué IP, qué gateway, qué DNS. Se llama *keyfile* y vive en `/etc/NetworkManager/system-connections/NOMBRE.nmconnection`.
**Por qué permisos `600` de root:** un perfil puede contener contraseñas de Wi-Fi o de VPN.
**Qué es `nmcli`:** el comando de NetworkManager. En modo lectura no necesita `sudo`; para cambiar, sí.
**Qué es DHCP:** el servidor de la red reparte las IPs automáticamente. Es lo que hace el hipervisor con `10.0.2.15`. Lo contrario es la IP fija (`manual`).
**Qué es host-only:** una red privada entre tu PC y la VM, sin salida a internet. Por eso **no lleva gateway**.
**Qué es `nmtui`:** la misma configuración con menús de texto, para el que se pierde con `nmcli`.

---

## Lo que hay que decir sí o sí

**El ciclo `add` → `modify` → `up`.** Decirlo despacio y repetirlo cuando falle: *"`modify` edita el archivo. **No pasa nada** hasta que hagan `up`. El error número uno de hoy va a ser modificar y no aplicar, y va a parecer que el comando no sirvió"*.

**No se pone gateway en la red host-only.** Si se lo ponen, aparece una segunda ruta por defecto y la VM intenta salir a internet por una red que no lleva a ningún lado. Frase: *"gateway solo hay uno, y ya lo tienen en la tarjeta del NAT"*.

**`/etc/resolv.conf` lo escribe NetworkManager.** Ya lo vieron en el bloque anterior. Acá se comprueba: se cambia el DNS con `nmcli`, se hace `up`, y el archivo cambia solo.

**Device vs connection, otra vez.** Acá se ve claro por primera vez: la tarjeta se llama `enp0s8` y el perfil se llama `lab`. En `nmcli device status` quedan uno al lado del otro.

## Qué señalar en cada parte

**El keyfile** — que lo abran una vez antes de crear el suyo. Cada sección del archivo corresponde a un grupo de propiedades de `nmcli`. Nadie edita ese archivo a mano hoy; se mira para entender qué escribe `nmcli`.

**El comando de `add`** — leerlo de izquierda a derecha nombrando cada pieza. Es largo pero es **una sola línea**; que no lo partan.

**`+` y `-`** — agregar y quitar de una lista sin reescribirla entera. Se usa con DNS y con rutas.

**`hostnamectl set-hostname`** — aviso para que no se asusten: el prompt va a seguir diciendo `rhel01` aunque el nombre completo sea `rhel01.lab.local`, porque el prompt solo muestra la parte anterior al primer punto. Se comprueba con `hostname`, no mirando el prompt.

**Los alias en `/etc/hosts`** — `servidor-nfs` y `servidor-smb` apuntan a la propia VM. Sirven para que los labs de los días siguientes usen nombres en vez de IPs, como en un servidor real, sin necesidad de montar un DNS.

## El riesgo del bloque

En el Lab 3.1 se **apaga la VM** para agregar la tarjeta. Avisar antes: la sesión SSH se va a cortar, es lo esperado, y hay que volver a entrar después.

En los labs 3.2 y 3.3, **todo se hace desde la sesión `ssh -p 2222`** (la del NAT). Si alguien entra por `192.168.56.10` y hace `nmcli con up lab`, se corta a sí mismo la sesión. Decirlo antes de que pase.
