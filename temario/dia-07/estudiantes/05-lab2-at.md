# Lab 2 — `at`

Vamos a programar algo para dentro de dos minutos, algo para las 17:30, y borrar lo que no queremos.

---

## Parte 1 — Instalar y activar

**¿Está `at` en este servidor?**

```bash
rpm -q at || sudo dnf install -y at
sudo systemctl enable --now atd
systemctl is-active atd
```

**Comprobar:** `rpm -q` muestra `at-3.1.23-...` o `dnf` lo instala. Al final, `active`.

---

## Parte 2 — Un trabajo para dentro de dos minutos

**¿Cómo le doy a `at` varios comandos?**

```bash
at now + 2 minutes
```

Se abre el prompt `at>`. Escribir las dos líneas y cerrar con `Ctrl+D`:

```
at> echo "Trabajo at ejecutado: $(date)" >> /home/student/at-prueba.txt
at> logger -t at-demo "trabajo de at ejecutado por $USER"
at> <Ctrl+D>
```

**Comprobar:**
```
warning: commands will be executed using /bin/sh
job 1 at Tue Sep 16 11:02:00 2026
```

---

## Parte 3 — Programar en una línea, listar y borrar

**¿Cómo veo qué hay pendiente y cómo cancelo uno?**

```bash
echo "logger -t at-demo 'recordatorio de las 17:30'" | at 17:30
atq
atrm 2
atq
```

Ahora ustedes: programen para dentro de 5 minutos `date >> /home/student/at-mio.txt` (en una línea, con `echo ... | at`) y muestren `atq`. Foto.

**Comprobar:**
```
warning: commands will be executed using /bin/sh
job 2 at Tue Sep 16 17:30:00 2026
1	Tue Sep 16 11:02:00 2026 a student
2	Tue Sep 16 17:30:00 2026 a student
1	Tue Sep 16 11:02:00 2026 a student
```
Si ya pasaron las 17:30, el trabajo 2 queda para mañana.

---

## Parte 4 — Cuando pasen los dos minutos

**¿Se ejecutó el trabajo 1?**

```bash
cat ~/at-prueba.txt
sudo journalctl -t at-demo -n 1 --no-pager
```

**Comprobar:**
```
Trabajo at ejecutado: Tue Sep 16 11:02:00 AM EST 2026
Sep 16 11:02:00 rhel01 at-demo[7001]: trabajo de at ejecutado por student
```
