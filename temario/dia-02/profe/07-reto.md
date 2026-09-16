# Reto — generador y solución

Correr el generador y la solución en la VM antes de clase; anotar el `md5sum` y las respuestas. En clase se compara con lo que peguen.

## Generador (pegar en el chat, como bloque de código)

```bash
mkdir -p ~/reto && cat > ~/reto/genera-reto.sh <<'FIN'
#!/bin/bash
# genera-reto.sh: log de 200 lineas con semilla fija (todos obtienen el mismo archivo)
RANDOM=2026
niveles=(INFO INFO WARN ERROR FAIL)
usuarios=(ana carlos pedro root backup)
servicios=(sshd httpd crond nfs-server backup.sh)
mensajes=("Sesion iniciada" "Sesion cerrada" "Conexion rechazada" "Archivo no encontrado" "Tarea completada" "Tiempo de espera agotado" "Permiso denegado" "Disco al 90 por ciento")
for i in $(seq 1 200); do
  printf "2026-04-%02d %02d:%02d:%02d %s %s %s %s\n" \
    $((RANDOM % 30 + 1)) $((RANDOM % 24)) $((RANDOM % 60)) $((RANDOM % 60)) \
    "${niveles[RANDOM % 5]}" "${usuarios[RANDOM % 5]}" \
    "${servicios[RANDOM % 5]}" "${mensajes[RANDOM % 8]}"
done > ~/reto/acceso.log
FIN
bash ~/reto/genera-reto.sh && wc -l ~/reto/acceso.log && md5sum ~/reto/acceso.log
```

## Solución

```bash
# 1
grep -cw FAIL ~/reto/acceso.log

# 2
grep -w WARN ~/reto/acceso.log | tail -1

# 3
grep -w ana ~/reto/acceso.log | sort -k1,2 > ~/reto/ana.log
wc -l ~/reto/ana.log
cut -d' ' -f4 ~/reto/ana.log | sort -u        # tiene que dar solo: ana

# 4
mkdir -p ~/backups
tar -czf ~/backups/reto-$(date +%F).tar.gz -C ~ reto
tar -tzf ~/backups/reto-$(date +%F).tar.gz

# 5
ln -s /home/student/backups/reto-$(date +%F).tar.gz ~/ultimo-backup
ls -l ~/ultimo-backup
cd /tmp && tar -tzf ~/ultimo-backup && cd

# Bonus
grep -Ew "ERROR|FAIL" ~/reto/acceso.log | cut -d' ' -f5 | sort | uniq -c | sort -rn | head -1
grep -w WARN ~/reto/acceso.log | sort -k1,2 | tail -1
```

## Entrega
```bash
grep -cw FAIL ~/reto/acceso.log; grep -w WARN ~/reto/acceso.log | tail -1; wc -l ~/reto/ana.log; ls -l ~/ultimo-backup; tar -tzf ~/ultimo-backup
```

Errores que se ven: enlace con ruta relativa desde otra carpeta (en rojo, `tar` dice `No such file`); fecha escrita a mano en el nombre; `grep ana` sin `-w` (funciona igual acá, pero comentarlo).
