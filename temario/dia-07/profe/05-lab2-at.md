# Lab 2 — `at` (comandos)

## Parte 1 — Instalar y activar
```bash
rpm -q at || sudo dnf install -y at
sudo systemctl enable --now atd
systemctl is-active atd
```
Si `dnf` falla con `Failed to download metadata`: la VM no tiene suscripción o red. `sudo subscription-manager status`, `sudo dnf repolist`.

## Parte 2 — Un trabajo para dentro de dos minutos
```bash
at now + 2 minutes
```
En el prompt `at>`, una línea por vez, y `Ctrl+D` al final:
```
echo "Trabajo at ejecutado: $(date)" >> /home/student/at-prueba.txt
logger -t at-demo "trabajo de at ejecutado por $USER"
```
Si alguien se queda trabado en `at>`: `Ctrl+D`. Si lo escribió mal, `Ctrl+C` cancela sin crear el trabajo.

## Parte 3 — Programar en una línea, listar y borrar
```bash
echo "logger -t at-demo 'recordatorio de las 17:30'" | at 17:30
atq
atrm 2
atq
```
Ellos:
```bash
echo "date >> /home/student/at-mio.txt" | at now + 5 minutes
atq
```
Los números de trabajo de cada VM pueden ser distintos: `atrm` con el número que muestre **su** `atq`.

## Parte 4 — Cuando pasen los dos minutos
```bash
cat ~/at-prueba.txt
sudo journalctl -t at-demo -n 1 --no-pager
```
Se puede hacer al empezar el Lab 3 si todavía no pasaron los dos minutos.
