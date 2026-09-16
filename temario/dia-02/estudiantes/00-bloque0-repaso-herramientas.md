# 0 — Repaso y herramientas

## Conectarse

```bash
ssh -p 2222 student@localhost
hostname && cat /etc/redhat-release && uname -m
```

## Confirmar el registro

```bash
sudo dnf repolist
```

## Herramientas de hoy

| Herramienta | Para qué | Dónde se usa |
|---|---|---|
| `tree` | Ver directorios como diagrama | Lab 2.1 |
| `vim-enhanced` | Colores y `vimtutor` en vim | Lab 6.1 |
| `bash-completion` | Tab completa subcomandos | Todo el día |
| `zip` / `unzip` | Formato zip | Lab 5.1 |
| `bzip2` | Compresor | Lab 5.1 |
| `nano` | Editor alternativo | Lab 6.1 |

Si a alguien le falta alguna:

```bash
sudo dnf install -y tree vim-enhanced bash-completion zip unzip bzip2 nano
```
