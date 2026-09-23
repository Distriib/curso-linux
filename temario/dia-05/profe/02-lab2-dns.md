# Lab — Resolución de nombres (comandos)

**Qué es `dig`:** el comando para preguntarle al DNS a mano. Viene en el paquete `bind-utils`, instalado en el primer lab del día.
**Qué es un registro `A`:** la línea del DNS que dice "este nombre tiene esta IP versión 4". Hay otros tipos: `AAAA` (IPv6), `MX` (correo), `CNAME` (alias).
**Qué es el TTL:** el número antes de `IN A` (por ejemplo `300`). Son los segundos que se puede guardar en caché esa respuesta.
**Qué es `getent hosts`:** pregunta un nombre **como lo haría cualquier programa**, siguiendo el orden de `/etc/nsswitch.conf`: primero `/etc/hosts`, después el DNS.
**Qué es la consulta inversa (`-x`):** de IP a nombre, al revés de lo normal.

## Parte 1
```bash
dig redhat.com
dig +short redhat.com
dig google.com
```

## Parte 2
```bash
dig @8.8.8.8 +short redhat.com
dig -x 8.8.8.8 +short
```

## Parte 3
```bash
getent hosts localhost
getent hosts rhel01
getent hosts redhat.com
```

## Parte 4
```bash
echo "127.0.0.1 redhat.com" | sudo tee -a /etc/hosts
getent hosts redhat.com
dig +short redhat.com
sudo vim /etc/hosts
```
`G`, `dd`, `:wq`
```bash
getent hosts redhat.com
```

---

## Qué señalar

- **Parte 1:** las tres cosas de la salida larga: `status: NOERROR` (el nombre existe), la `ANSWER SECTION`, y `SERVER:` al final. Si dijera `NXDOMAIN`, el nombre no existe; si `SERVFAIL`, el servidor no pudo contestar.
- **Parte 2:** esta es **la** técnica de diagnóstico de DNS. Frase: *"si `@8.8.8.8` responde y sin `@` no, el problema es el DNS que tienen configurado, no internet. Eso les ahorra media hora de buscar donde no es"*.
- **Parte 3:** `rhel01` no está en `/etc/hosts` y aun así responde: lo resuelve el propio sistema por el nombre de la máquina. Puede salir con una dirección `fe80::` en vez de la IPv4; es normal, `getent` pregunta primero por IPv6.
- **Parte 4:** el `tee -a` agrega al final del archivo (el Día 2 vieron por qué `sudo echo >` no sirve). Que **verifiquen que la línea quedó borrada**: si queda, `redhat.com` apunta a `127.0.0.1` el resto del curso y algo va a fallar raro más adelante.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `dig: command not found` | Falta `bind-utils`; se instaló en el primer lab |
| `dig` tarda 10 segundos y falla | El DNS configurado no responde. Probar con `@8.8.8.8` |
| Borraron la línea equivocada de `/etc/hosts` | Que dejen las dos primeras (`127.0.0.1 localhost...` y `::1 localhost...`). Sin ellas fallan cosas raras |
| `getent hosts rhel01` devuelve una dirección larga con dos puntos | Es la IPv6 local. Normal |
