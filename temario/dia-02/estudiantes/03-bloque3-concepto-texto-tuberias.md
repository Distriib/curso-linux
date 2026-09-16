# 3 — Texto, tuberías y redirección

## Ver texto

```bash
head -3 /etc/passwd
tail -3 /etc/passwd
wc -l /etc/passwd
less /etc/services
```

| Comando | Qué hace |
|---|---|
| `cat archivo` | muestra todo (solo para archivos cortos) |
| `less archivo` | pagina: `Espacio` avanza, `/texto` busca, `n` siguiente, `q` sale |
| `head -n 5` / `tail -n 5` | primeras / últimas líneas |
| `tail -f archivo` | se queda mirando el archivo y muestra lo que se agregue (un log en vivo) |
| `wc -l` | cuenta líneas |

---

## `grep` — buscar líneas

```bash
grep student /etc/passwd
grep -n bash /etc/passwd
grep -c nologin /etc/passwd
grep -v nologin /etc/passwd
grep -i STUDENT /etc/passwd
```

| Opción | Qué hace |
|---|---|
| `-i` | ignora mayúsculas |
| `-v` | invierte: las que **no** coinciden |
| `-n` | número de línea |
| `-c` | cuenta |
| `-w` | palabra completa |
| `-E` | expresiones regulares extendidas |

### Expresiones regulares (con `-E`)

| Símbolo | Significa |
|---|---|
| `^texto` | empieza con |
| `texto$` | termina con |
| `A\|B` | A o B |
| `.` | cualquier carácter |
| `[0-9]` | un dígito |

```bash
grep -E "^student|^root" /etc/passwd
grep -E "bash$" /etc/passwd
sudo grep -Ev "^#|^$" /etc/ssh/sshd_config
```

---

## Cortar, ordenar, contar

```bash
cut -d: -f1 /etc/passwd
cut -d: -f7 /etc/passwd | sort
cut -d: -f7 /etc/passwd | sort | uniq -c
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```

| Comando | Qué hace |
|---|---|
| `cut -d: -f1` | campo 1, separado por `:` |
| `cut -d' ' -f3` | campo 3, separado por espacio |
| `sort` | ordena; `-n` numérico, `-r` inverso, `-u` sin repetidos |
| `uniq -c` | cuenta repetidos **consecutivos** (por eso va después de `sort`) |

> `sort | uniq -c | sort -rn` = "top de lo más repetido". El patrón más útil del día.

---

## Los tres flujos y la redirección

Cada programa tiene una **entrada** (lo que lee), una **salida** (lo que responde) y una **salida de error** (por donde se queja). Se pueden desviar.

```bash
cd /tmp
grep bash /etc/passwd > shells.txt
grep nologin /etc/passwd >> shells.txt
wc -l shells.txt
```

```bash
ls /etc/hostname /noexiste
ls /etc/hostname /noexiste > ok.txt 2> error.txt
cat ok.txt
cat error.txt
ls /noexiste 2> /dev/null
```

| Sintaxis | Efecto |
|---|---|
| `cmd > archivo` | salida a archivo (**lo pisa**) |
| `cmd >> archivo` | salida al final del archivo |
| `cmd 2> archivo` | errores a archivo |
| `cmd > a.txt 2> b.txt` | separados |
| `cmd 2> /dev/null` | tirar los errores |
| `cmd1 \| cmd2` | tubería: la salida de uno es la entrada del otro |
| `cmd \| tee archivo` | guarda una copia **y** sigue mostrando |

```bash
grep bash /etc/passwd | tee bash.txt | wc -l
```

### `sudo` y `>`

```bash
sudo echo "Servidor rhel01 - PGN" > /etc/motd
echo "Servidor rhel01 - PGN" | sudo tee /etc/motd
cat /etc/motd
```

```bash
rm shells.txt ok.txt error.txt bash.txt
cd
```
