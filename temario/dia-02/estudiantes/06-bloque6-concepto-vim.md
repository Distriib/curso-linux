# 6 — Editar con `vim`

## Dos estados

```
        i
  ┌──────────────►
MANDAR            ESCRIBIR
  ◄──────────────┘
        Esc
```

- **Mandar** (modo normal): las teclas son órdenes. Acá se arranca.
- **Escribir** (modo insertar): las teclas escriben texto. Se entra con `i`, se sale con `Esc`.
- Si no sabés en cuál estás: `Esc` `Esc`.

## Las seis teclas para sobrevivir

| Tecla | Hace |
|---|---|
| `i` | empezar a escribir |
| `Esc` | dejar de escribir, volver a mandar |
| `:wq` | guardar y salir |
| `:q!` | salir **sin** guardar |
| `dd` | borrar la línea |
| `u` | deshacer |

## Primera vez

```bash
vim ~/prueba.txt
```
`i` → escribir dos líneas → `Esc` → `:wq` `Enter`

```bash
cat ~/prueba.txt
vim ~/prueba.txt
```
`dd` → `u` → `:q!` `Enter`

## Las que ahorran tiempo

| Tecla | Hace |
|---|---|
| `/texto` `Enter` | buscar; `n` siguiente, `N` anterior; `:noh` quita el resaltado |
| `:set nu` | mostrar números de línea |
| `gg` / `G` | ir al inicio / al final |
| `o` | nueva línea abajo (y ya escribiendo) |
| `yy` / `p` | copiar línea / pegar abajo |
| `x` | borrar un carácter |
| `Ctrl+R` | rehacer |
| `:%s/viejo/nuevo/g` | reemplazar en todo el archivo |

## Si algo sale mal

- Pantalla rara, no responde → `Esc` `Esc` `:q!` `Enter`. El archivo no cambia hasta que hagas `:w`.
- `E37: No write since last change` → tenés cambios sin guardar: `:wq` para guardarlos, `:q!` para tirarlos.
- `E45: 'readonly' option is set` → abriste un archivo de root sin `sudo`: `:q!` y `sudo vim archivo`.

## `nano`, la alternativa

```bash
nano ~/prueba.txt
```
Escribís directo. `Ctrl+O` `Enter` guarda, `Ctrl+X` sale. Los atajos están abajo en pantalla.
