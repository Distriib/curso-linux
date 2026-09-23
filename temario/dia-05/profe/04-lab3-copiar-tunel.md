# Lab — Copiar archivos y túnel (comandos)

## Parte 1 — en el equipo del participante
```bash
echo "nota desde mi equipo" > nota.txt
scp nota.txt rhel01:empresa/documentos/
scp rhel01:/etc/hostname ./hostname-rhel01.txt
cat hostname-rhel01.txt
```
En la VM: `ls ~/empresa/documentos/`

## Parte 2 — dentro de la VM
```bash
rsync -avz --delete ~/empresa/ ~/empresa-copia/
touch ~/empresa/documentos/nuevo.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
rm ~/empresa/documentos/nuevo.txt
rsync -avz --delete ~/empresa/ ~/empresa-copia/
```

## Parte 3 — en la VM
```bash
cd ~/empresa
python3 -m http.server 8000
```

## Parte 4 — en el equipo del participante
```bash
curl -m 3 http://192.168.56.10:8000/
```

## Parte 5 — en el equipo del participante
```bash
ssh -N -L 8081:localhost:8000 rhel01
```
En otra ventana:
```bash
curl http://localhost:8081/
```
`Ctrl+C` en las dos.

---

## Qué señalar

- **Parte 1:** `-P` mayúscula en `scp`, `-p` minúscula en `ssh`. Si usan el alias no hace falta ninguno de los dos, que es la gracia.
- **Parte 2:** la segunda corrida solo manda `nuevo.txt`. Frase: *"por eso `rsync` es lo que se usa para respaldos: la primera vez tarda, las siguientes son segundos"*. Y la barra final: `~/empresa/` copia el contenido; sin barra crearía una carpeta adentro.
- **Parte 3:** `python3` viene instalado. Se usa el puerto 8000 y no el 80 justamente porque el 80 **sí** está publicado por el reenvío de puertos del Día 1 y no serviría para demostrar el túnel.
- **Parte 4:** tiene que fallar. Es la mitad de la demostración.
- **Parte 5:** leer el comando: `-L 8081:localhost:8000` es *"mi puerto 8081 → el 8000 visto desde el servidor"*. `-N` = no abrir shell, solo el túnel. Frase de cierre: *"así se llega a una consola de administración o a una base de datos sin abrir un puerto más en el firewall. Es de lo más usado y de lo menos enseñado"*.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `scp: ... No such file or directory` | La carpeta destino no existe en la VM |
| `rsync: command not found` | Se instaló en el primer lab del día; repetir el `dnf install` |
| `rsync` copió todo de nuevo la segunda vez | Pusieron la ruta sin barra final, o escribieron otro destino |
| `Address already in use` en el puerto 8000 | Ya lo tienen corriendo en otra ventana |
| El túnel "no hace nada" | Correcto: `-N` no abre shell. Se usa desde **otra** ventana |
| `curl` en PowerShell devuelve algo raro | `curl` ahí es otro comando; usar `curl.exe` |
