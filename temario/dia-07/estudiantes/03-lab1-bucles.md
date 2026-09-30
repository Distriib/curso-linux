# Lab 3.1 — Bucles sobre un log

Vamos a fabricar un log de 10 líneas con `for` y después leerlo con bucles. Todo en `~/empresa/logs`. Donde dice **Ahora ustedes**, el comando no está escrito: hay que resolverlo y mandar foto. La solución está al final de la hoja.

| Nivel | Usuario | Líneas | Hora |
|---|---|---|---|
| `INFO` | `ana` | 5 | `08:01` a `08:05` |
| `ERROR` | `carlos` | 3 | `08:11` a `08:13` |
| `WARN` | `pedro` | 2 | `08:21` a `08:22` |

---

## Parte 1 — `for` sobre una lista de palabras

**¿Cómo pregunto el estado de tres servicios sin escribir tres comandos?**

```bash
for svc in sshd crond chronyd; do echo "$svc: $(systemctl is-active $svc)"; done
```

Ahora ustedes: con `firewalld`, `atd` y `httpd`. Foto.

**Comprobar:**
```
sshd: active
crond: active
chronyd: active
```

---

## Parte 2 — `for` sobre números: fabricar el log

**¿Cómo genero cinco líneas parecidas que solo cambian en un número?**

```bash
cd ~/empresa/logs
rm -f app.log
for i in {1..5}; do echo "2026-09-01 08:0$i INFO usuario=ana accion=login" >> app.log; done
```

Ahora ustedes: las líneas de `ERROR` (`carlos`, `08:1$i`, `{1..3}`) y las de `WARN` (`pedro`, `08:2$i`, `accion=disco`, `{1..2}`), según la tabla. Después `cat app.log` y `wc -l app.log`. Foto.

**Comprobar:**
```
2026-09-01 08:01 INFO usuario=ana accion=login
...
2026-09-01 08:13 ERROR usuario=carlos accion=login
2026-09-01 08:21 WARN usuario=pedro accion=disco
2026-09-01 08:22 WARN usuario=pedro accion=disco
10 app.log
```

---

## Parte 3 — Contar

**¿Cuántas líneas hay de cada nivel?**

```bash
grep -c ERROR app.log
cut -d' ' -f3 app.log | sort | uniq -c
```

Ahora ustedes: lo mismo por usuario (campo 4). Foto.

**Comprobar:**
```
3
      3 ERROR
      5 INFO
      2 WARN
```

---

## Parte 4 — `for` sobre archivos

**¿Cuántas líneas tiene cada `.log` de la carpeta?**

```bash
for f in ~/empresa/logs/*.log; do echo "$f: $(wc -l < "$f") líneas"; done
```

**Comprobar:**
```
/home/student/empresa/logs/app.log: 10 líneas
/home/student/empresa/logs/servidor.log: 300 líneas
```
`*.log` se convierte en la lista de archivos que terminan así. `servidor.log` es el del Día 2: si ya no lo tienen, sale solo la línea de `app.log` y está bien igual.

---

## Parte 5 — `while read`: una línea por vez

**¿Cómo separo cada línea en sus campos?**

```bash
while read -r fecha hora nivel resto; do
    echo "$hora $nivel -> $resto"
done < app.log
```

Ahora ustedes: cambien las variables a `fecha hora nivel usuario accion` y muestren `"$usuario tuvo $nivel"`. Foto.

**Comprobar:**
```
08:01 INFO -> usuario=ana accion=login
...
08:22 WARN -> usuario=pedro accion=disco
```
La última variable (`resto`) se queda con todo lo que sobra de la línea.

```bash
cd
```

---

# Solución — todos los comandos

```bash
# Parte 1 — for sobre palabras
for svc in sshd crond chronyd; do echo "$svc: $(systemctl is-active $svc)"; done
```
**Parte 1 (Ahora ustedes)** — los otros tres servicios:
```bash
for svc in firewalld atd httpd; do echo "$svc: $(systemctl is-active $svc)"; done
```
`firewalld` dice `active`. `httpd` y, según cómo se instaló la VM, también `atd`, van a decir `inactive` o `unknown`: todavía no están instalados. Es correcto — `at` se instala en el Lab 5.2 y `httpd` recién el Día 8.

```bash
# Parte 2 — fabricar el log
cd ~/empresa/logs
rm -f app.log
for i in {1..5}; do echo "2026-09-01 08:0$i INFO usuario=ana accion=login" >> app.log; done
```
**Parte 2 (Ahora ustedes)** — las líneas de ERROR y de WARN:
```bash
for i in {1..3}; do echo "2026-09-01 08:1$i ERROR usuario=carlos accion=login" >> app.log; done
for i in {1..2}; do echo "2026-09-01 08:2$i WARN usuario=pedro accion=disco" >> app.log; done
cat app.log
wc -l app.log
```

```bash
# Parte 3 — contar
grep -c ERROR app.log
cut -d' ' -f3 app.log | sort | uniq -c
```
**Parte 3 (Ahora ustedes)** — lo mismo por usuario, que es el campo 4:
```bash
cut -d' ' -f4 app.log | sort | uniq -c
```
```
      5 usuario=ana
      3 usuario=carlos
      2 usuario=pedro
```
Salen en orden alfabético, no por cantidad: los ordenó el `sort` antes de que `uniq` los contara.

```bash
# Parte 4 — for sobre archivos
for f in ~/empresa/logs/*.log; do echo "$f: $(wc -l < "$f") líneas"; done

# Parte 5 — while read
while read -r fecha hora nivel resto; do
    echo "$hora $nivel -> $resto"
done < app.log
```
**Parte 5 (Ahora ustedes)** — cinco variables en vez de cuatro:
```bash
while read -r fecha hora nivel usuario accion; do
    echo "$usuario tuvo $nivel"
done < app.log
```
Ahora `usuario` se queda con `usuario=ana` y `accion` con `accion=login`: al haber una variable más, el reparto cambia.

```bash
cd
```
