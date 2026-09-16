# 4 — Buscar archivos: `find` y `locate`

## `find` — buscar recorriendo el disco

```
find  DÓNDE  CRITERIOS  [ACCIÓN]
```

```bash
find ~/empresa -name "*.txt"
find ~/empresa -name "*.txt" | wc -l
find ~/empresa -type d
find ~/empresa -empty
```

### Criterios

| Criterio | Busca |
|---|---|
| `-name "*.conf"` | por nombre (con comillas) |
| `-iname "*.CONF"` | por nombre, sin distinguir mayúsculas |
| `-type f` / `d` / `l` | archivo / directorio / enlace simbólico |
| `-size +100k` / `-size -1M` | más grande que / más chico que |
| `-mtime -1` / `-mtime +7` | modificado hace menos de 1 día / más de 7 |
| `-mmin -60` | modificado en los últimos 60 minutos |
| `-user student` | del usuario |
| `-empty` | vacío |
| `-maxdepth 1` | sin bajar de nivel |

```bash
find /etc -name "*.conf"
sudo find /etc -name "*.conf" | wc -l
sudo find /etc -type f -size +100k
find ~/empresa -mmin -120
```

### Acciones

| Acción | Hace |
|---|---|
| (ninguna) | muestra la ruta |
| `-exec cmd {} \;` | ejecuta `cmd` **una vez por cada** resultado |
| `-exec cmd {} +` | ejecuta `cmd` **una sola vez** con todos los resultados |
| `-delete` | borra. Siempre al **final**, siempre después de probar sin él |

```bash
find ~/empresa -name "*.txt" -exec wc -l {} \;
find ~/empresa -name "*.txt" -exec wc -l {} +
```

> `find` sin `sudo` en carpetas del sistema tira `Permission denied`. O `sudo find`, o `2> /dev/null`.

---

## `locate` — buscar en una base de datos

```bash
sudo dnf install -y mlocate
locate servidor.log
sudo updatedb
locate servidor.log
locate -i SERVIDOR
```

| | `find` | `locate` |
|---|---|---|
| Cómo busca | recorre el disco ahora | consulta una lista hecha antes |
| Velocidad | lenta en carpetas grandes | instantánea |
| Ve archivos nuevos | sí | no hasta el próximo `updatedb` (corre solo cada día) |
| Criterios | nombre, tipo, tamaño, fecha, dueño… | solo nombre |

---


