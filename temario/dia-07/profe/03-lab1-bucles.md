# Lab 1 — Bucles sobre un log (comandos)

## Parte 1 — `for` sobre una lista de palabras
```bash
for svc in sshd crond chronyd; do echo "$svc: $(systemctl is-active $svc)"; done
```
Ellos:
```bash
for svc in firewalld atd httpd; do echo "$svc: $(systemctl is-active $svc)"; done
```
`atd` y `httpd` pueden dar `inactive`: es un dato, no un error. `atd` se activa en el Bloque 5.

## Parte 2 — `for` sobre números: fabricar el log
```bash
cd ~/empresa/logs
rm -f app.log
for i in {1..5}; do echo "2026-09-01 08:0$i INFO usuario=ana accion=login" >> app.log; done
```
Ellos:
```bash
for i in {1..3}; do echo "2026-09-01 08:1$i ERROR usuario=carlos accion=login" >> app.log; done
for i in {1..2}; do echo "2026-09-01 08:2$i WARN usuario=pedro accion=disco" >> app.log; done
cat app.log
wc -l app.log
```
Si a alguien le da más de 10 líneas, ejecutó un `for` dos veces: `rm -f app.log` y los tres `for` de nuevo.

## Parte 3 — Contar
```bash
grep -c ERROR app.log
cut -d' ' -f3 app.log | sort | uniq -c
```
Ellos:
```bash
cut -d' ' -f4 app.log | sort | uniq -c
```

## Parte 4 — `for` sobre archivos
```bash
for f in ~/empresa/logs/*.log; do echo "$f: $(wc -l < "$f") líneas"; done
```
Quien no tenga `servidor.log` (Día 2 incompleto) ve solo `app.log`.

## Parte 5 — `while read`: una línea por vez
```bash
while read -r fecha hora nivel resto; do
    echo "$hora $nivel -> $resto"
done < app.log
```
Ellos:
```bash
while read -r fecha hora nivel usuario accion; do
    echo "$usuario tuvo $nivel"
done < app.log
cd
```
