# 5 — Respaldos: `tar` y compresión

**Empaquetar** = juntar muchos archivos en uno (`tar`). **Comprimir** = hacerlo más chico (`gzip`). Un respaldo es las dos cosas: `.tar.gz`.

## Las letras de `tar`

| Letra | Significa |
|---|---|
| `c` | **c**rear |
| `x` | e**x**traer |
| `t` | lis**t**ar |
| `z` | con gzip |
| `f archivo` | el archivo `.tar.gz` — va **última** |
| `-C carpeta` | ir a esa carpeta antes de actuar |
| `v` | mostrar cada archivo mientras trabaja |

## Crear, listar, extraer, comprobar

```bash
cd /tmp
tar -czf empresa.tar.gz -C ~ empresa
ls -lh empresa.tar.gz
```

```bash
tar -tzf empresa.tar.gz | head -5
```

```bash
mkdir restaurar
tar -xzf empresa.tar.gz -C restaurar
ls restaurar/empresa
```

```bash
diff -r ~/empresa restaurar/empresa && echo "IGUAL"
```

## Compresores

| | Velocidad | Tamaño | Extensión |
|---|---|---|---|
| `gzip` | rápido | normal | `.gz` (`tar -z`) |
| `bzip2` | medio | más chico | `.bz2` (`tar -j`) |
| `xz` | lento | el más chico | `.xz` (`tar -J`) |

`zip` comprime y empaqueta en uno; es el formato de Windows. No guarda permisos.

```bash
rm -r empresa.tar.gz restaurar
cd
```
