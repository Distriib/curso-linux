# Reto — Solución

```bash
# 1. Grupo, usuarios, contraseñas, plazos
sudo groupadd -g 3010 expediente
sudo useradd -m -u 2010 -G expediente -c "Maria Perez - Expediente" -e 2026-12-31 maria
sudo useradd -m -u 2011 -G expediente -c "Jorge Diaz - Expediente"  -e 2026-12-31 jorge
sudo useradd -m -u 2012 -c "Auditor externo" auditor_ext
echo 'Pgn.2026' | sudo passwd --stdin maria
echo 'Pgn.2026' | sudo passwd --stdin jorge
echo 'Pgn.2026' | sudo passwd --stdin auditor_ext
sudo chage -M 60 -W 10 maria
sudo chage -M 60 -W 10 jorge

# 2. Carpeta compartida con herencia de grupo
sudo mkdir -p /srv/expediente
sudo chown root:expediente /srv/expediente
sudo chmod 2770 /srv/expediente

# 3. Solo lectura para auditor_ext, presente y futuro
sudo setfacl -m u:auditor_ext:rx /srv/expediente
sudo setfacl -d -m u:auditor_ext:rx /srv/expediente

# 4. Regla sudo limitada
sudo visudo -f /etc/sudoers.d/expediente
```
Contenido de `/etc/sudoers.d/expediente`:
```
%expediente ALL=(root) /usr/bin/systemctl status chronyd, /usr/bin/systemctl restart chronyd
```

## Verificación
```bash
id maria; id jorge; id auditor_ext
sudo chage -l maria
ls -ld /srv/expediente
sudo -u maria bash -c 'echo caso > /srv/expediente/caso-001.txt'; sudo ls -l /srv/expediente
sudo -u auditor_ext cat /srv/expediente/caso-001.txt
sudo -u auditor_ext touch /srv/expediente/x.txt
sudo -u laura ls /srv/expediente
sudo -l -U jorge | tail -1
sudo -l -U laura
sudo visudo -c
```

## Errores que se ven

- Sin el `-d` en la ACL: el auditor lee `caso-001.txt` pero no lo que se cree después.
- `chmod 770` sin el `2`: los archivos nacen con el grupo privado de maria, no con `expediente`.
- Agregar a `auditor_ext` al grupo: viola el requisito (tendría escritura).
- `/bin/systemctl` en vez de `/usr/bin/systemctl`: funciona (es enlace), aceptar.
- Poner el `-e` en `chage -E` en vez de `useradd -e`: equivalente, aceptar.
