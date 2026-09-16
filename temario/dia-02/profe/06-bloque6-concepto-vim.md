# 6 — Editar con `vim` (guía del instructor)

Cinco minutos. Este bloque es un guion de teclas: seguilo tal cual, tecla
por tecla, y hacelo **vos primero en pantalla** antes de que ellos toquen
nada.

**Qué es `vim`:** un editor de texto que corre dentro de la terminal, sin mouse. Está en todo Linux, hasta en modo rescate, y es lo único que garantiza el examen RHCSA. Por eso se enseña este y no otro.
**Por qué asusta:** porque tiene *modos*. Al abrirlo, las teclas no
escriben — son órdenes. La gente tipea, no pasa nada (o pasan cosas
raras), y no sabe cómo salir.
**`vim-minimal` vs `vim-enhanced`:** RHEL trae siempre el mínimo (`vi`).
El `enhanced` (instalado en el Bloque 0) agrega colores y `vimtutor`.

## El guion (5 min)

**1. (30 s) La idea.** Decir, mirándolos: *"vim tiene dos estados:
escribir y mandar. Arranca en mandar. `i` los pasa a escribir. `Esc` los
devuelve a mandar. Cuando no sepan en cuál están: `Esc` dos veces."*

**2. (30 s) Las seis teclas.** *"Hoy necesitan seis: `i`, `Esc`, `:wq`,
`:q!`, `dd`, `u`. El resto es velocidad, no necesidad."* Señalar la tabla
en pantalla.

**3. (60 s) Demo tuya.** Tecla por tecla, diciendo cada una en voz alta:

```bash
vim ~/prueba.txt
```
- `i` → abajo a la izquierda aparece `-- INSERT --`. Decir: *"ahora estoy
  escribiendo"*.
- Escribir `hola` `Enter` `mundo`.
- `Esc` → desaparece el `-- INSERT --`. Decir: *"ahora estoy mandando"*.
- `:wq` `Enter` → decir: *"dos puntos abre la línea de órdenes; w es
  guardar, q es salir"*. Vuelve el prompt.

```bash
cat ~/prueba.txt
vim ~/prueba.txt
```
- `dd` → se borra la línea `hola`. Decir: *"d d, borra la línea"*.
- `u` → vuelve. Decir: *"u, deshacer"*.
- `:q!` `Enter` → decir: *"q con signo de admiración: salir sin guardar.
  Como no guardé, el archivo quedó igual"*.

**4. (90 s) Todos lo repiten.** *"Ahora ustedes: `vim ~/prueba.txt`,
escriban su nombre, guarden, salgan. Después vuelvan a entrar, borren
una línea con `dd`, y salgan con `:q!`."* **Nadie sigue hasta que todos
hayan vuelto al prompt las dos veces.** Si alguien está atrapado: `Esc`
`Esc` `:q!` `Enter`.

**5. (60 s) Las que ahorran tiempo.** Señalar la segunda tabla: *"`/`
busca como en `less` y en `man`; `:set nu` numera; `:%s/viejo/nuevo/g`
reemplaza en todo el archivo — eso lo van a usar en el lab."* No
demostrarlas; se practican en el lab.

## Qué decir si pasa

| Pasa | Qué decir |
|---|---|
| Escribió y no aparece nada / aparecen cosas raras | "Estás en mandar. `i` para escribir." |
| `:wq` aparece escrito en el texto | "Estabas en escribir. `Esc`, y después `:wq`." |
| `E37: No write since last change` | "Tenés cambios sin guardar. `:wq` los guarda, `:q!` los tira." |
| `E45: 'readonly'` | "Abriste un archivo de root sin `sudo`. `:q!` y `sudo vim archivo`." |
| Desapareció vim y volvió el prompt | Apretó `Ctrl+Z` (lo suspendió). `fg` lo trae de vuelta. |
| Pantalla congelada | `Ctrl+S`. `Ctrl+Q` la libera. |

## `nano`

Un minuto, solo mencionarlo: *"si vim se les hace imposible, `nano`
funciona como un editor normal: escriben, `Ctrl+O` guarda, `Ctrl+X`
sale. Pero en el examen no está garantizado; vim sí."*

## Para editar archivos del sistema

`sudo vim /etc/archivo`. Sin `sudo`, abre pero no deja guardar (`E45`).
Hoy no se edita nada de `/etc`; queda para más adelante.
