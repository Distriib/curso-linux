# Lab — Primer script

Vamos a dejar tres scripts en `~/bin` y ejecutarlos por nombre desde cualquier carpeta.

| Script | Qué hace |
|---|---|
| `hola.sh` | variables, comillas y `$( )` |
| `args.sh` | muestra lo que recibe |
| `reporte.sh` | pide un dato y arma un reporte del servidor |

---

## Parte 1 — Preparar las carpetas

**¿Dónde van los scripts propios y dónde los respaldos de hoy?**

```bash
mkdir -p ~/bin ~/empresa/logs
sudo mkdir -p /datos/backups
sudo chown student:student /datos/backups
echo "$PATH"
```

**Comprobar:**
```
/home/student/.local/bin:/home/student/bin:/usr/local/bin:/usr/bin:...
```
`/home/student/bin` está en el `PATH` aunque la carpeta recién exista.

---

## Parte 2 — Escribir el primer script

**¿Cómo se guarda un script sin abrir el editor?**

```bash
cat > ~/bin/hola.sh <<'EOF'
#!/bin/bash
# hola.sh - primer script: variables, comillas y $( )
NOMBRE="Procuraduria"
FECHA=$(date +%F)

echo "Hola, $NOMBRE. Hoy es $FECHA"
echo 'Con comillas simples no se reemplaza: $NOMBRE'
echo "Archivo de salida: ${NOMBRE}_reporte.txt"
echo "Cuenta: $((7 * 6))"
EOF
ls -l ~/bin/hola.sh
bash ~/bin/hola.sh
```

**Comprobar:**
```
-rw-rw-r--. 1 student student ... /home/student/bin/hola.sh
Hola, Procuraduria. Hoy es 2026-09-...
Con comillas simples no se reemplaza: $NOMBRE
Archivo de salida: Procuraduria_reporte.txt
Cuenta: 42
```
Sin `x` en los permisos, pero `bash hola.sh` lo ejecuta igual.

---

## Parte 3 — Permiso de ejecución y `PATH`

**¿Por qué `./hola.sh` no funciona todavía?**

```bash
cd ~/bin
./hola.sh
chmod +x hola.sh
./hola.sh
cd /tmp
hola.sh
cd
```

**Comprobar:** el primer `./hola.sh` da `bash: ./hola.sh: Permission denied`. Después del `chmod`, las cuatro líneas. Desde `/tmp`, `hola.sh` a secas también funciona: `~/bin` está en el `PATH`.

---

## Parte 4 — Lo que recibe un script

**¿Cómo sabe el script qué le pasaron?**

```bash
cat > ~/bin/args.sh <<'EOF'
#!/bin/bash
# args.sh - muestra lo que recibe
echo "Nombre del script: $0"
echo "Primer argumento:  $1"
echo "Segundo argumento: $2"
echo "Cantidad:          $#"
echo "Todos:             $@"
EOF
chmod +x ~/bin/args.sh
args.sh servidor01 "Sala de servidores" 42
```

Ahora ustedes: `args.sh` con su nombre, su área entre comillas y un número. Después lo mismo **sin** las comillas. Foto.

**Comprobar:**
```
Nombre del script: /home/student/bin/args.sh
Primer argumento:  servidor01
Segundo argumento: Sala de servidores
Cantidad:          3
Todos:             servidor01 Sala de servidores 42
```
Sin las comillas, `Cantidad` sube: cada palabra es un argumento.

---

## Parte 5 — El código de salida

**¿Cómo sabe el sistema si un comando salió bien?**

```bash
ls /etc/hostname; echo "código: $?"
ls /nada; echo "código: $?"
```

**Comprobar:**
```
/etc/hostname
código: 0
ls: cannot access '/nada': No such file or directory
código: 2
```
`0` = bien. Cualquier otro número = algo falló.

---

## Parte 6 — Un reporte con datos del servidor

**¿Cómo mete un script la salida de otros comandos en su texto?**

```bash
cat > ~/bin/reporte.sh <<'EOF'
#!/bin/bash
# reporte.sh - reporte breve del servidor
read -p "Nombre del técnico: " TECNICO
echo "=== Reporte de $(hostname) ==="
echo "Generado por: $TECNICO"
echo "Fecha:        $(date '+%F %T')"
echo "Kernel:       $(uname -r)"
echo "Encendido:    $(uptime -p)"
echo "Disco raíz:   $(df -h / | tail -1)"
EOF
chmod +x ~/bin/reporte.sh
reporte.sh
```

Ahora ustedes: con `vim ~/bin/reporte.sh`, agreguen al final la línea `echo "Conectados:   $(who | wc -l)"` y ejecútenlo otra vez. Foto.

**Comprobar:**
```
Nombre del técnico: Esteban
=== Reporte de rhel01 ===
Generado por: Esteban
Fecha:        2026-09-... ..:..:..
Kernel:       5.14.0-...
Encendido:    up ...
Disco raíz:   /dev/mapper/rhel-root  17G  ...  ...  ...% /
```
