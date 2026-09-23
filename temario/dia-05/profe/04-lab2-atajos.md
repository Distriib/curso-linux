# Lab — Alias y huella (comandos)

Todo en el equipo del participante.

## Parte 1
Mac/Linux:
```bash
vim ~/.ssh/config
```
Contenido:
```
Host rhel01
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_ed25519

Host rhel01-nat
    HostName localhost
    Port 2222
    User student
    IdentityFile ~/.ssh/id_ed25519
```
```bash
chmod 600 ~/.ssh/config
```
Windows: `notepad $env:USERPROFILE\.ssh\config` y pegar lo mismo.

## Parte 2
```bash
ssh rhel01 hostname
ssh rhel01-nat "ip -br a show enp0s8"
ssh rhel01
exit
```

## Parte 3
```bash
ssh-keygen -R 192.168.56.10
ssh rhel01 hostname
```

---

## Qué señalar

- **Parte 1:** en UTM, el `HostName` del alias `rhel01` es la IP del rango host-only del Mac, no `192.168.56.10`.
- **Parte 2:** el alias también sirve para `scp` y `rsync` (`scp archivo rhel01:/tmp/`), que se usa en el lab siguiente.
- **Parte 3:** que **comparen la huella** con la que vieron en la VM. No es un trámite: es lo único que distingue "el servidor se reinstaló" de "alguien se está haciendo pasar por el servidor".
  Aviso: si además borran la entrada de `[localhost]:2222`, la próxima conexión por `rhel01-nat` también va a preguntar. Se responde `yes` una vez.

## Errores que vas a ver

| Pasa | Por qué |
|---|---|
| `Bad owner or permissions on ~/.ssh/config` | Falta `chmod 600` |
| En Windows el archivo quedó como `config.txt` | Notepad agregó la extensión. Que lo renombren |
| El alias no funciona | Mal la sangría del archivo, o `Host` con mayúscula equivocada |
| `ssh rhel01` no llega | La VM tiene otra IP, o el perfil `lab` está caído. `ip -br a` desde la sesión del NAT |
