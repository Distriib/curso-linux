# Lab — Servidor NFS (comandos)

En tu VM (UTM) reemplazá `192.168.56.0/24` por tu red host-only si es otra; ellos usan `192.168.56.0/24`.

## Parte 1 — Instalar y arrancar
```bash
sudo dnf install -y nfs-utils
sudo systemctl enable --now nfs-server
systemctl status nfs-server --no-pager | head -4
sudo ss -tln | grep 2049
```
Decir: "`active (exited)` es normal: NFS vive en el kernel. Por eso el 2049 tampoco muestra un proceso."

## Parte 2 — Las carpetas
```bash
sudo mkdir -p /srv/nfs/compartido
sudo chown student:student /srv/nfs/compartido
sudo mkdir -p /srv/nfs/lectura
echo "Solo lectura desde NFS - rhel01" | sudo tee /srv/nfs/lectura/README.txt
ls -ld /srv/nfs/*
cat /srv/nfs/lectura/README.txt
```
Decir: "`compartido` es de `student` (UID 1000) para que el mismo UID pueda escribir desde el cliente sin ser root".

## Parte 3 — `/etc/exports`
```bash
sudo vim /etc/exports
```
```
/srv/nfs/compartido    192.168.56.0/24(rw,sync,no_root_squash)
/srv/nfs/lectura       127.0.0.1(ro,sync)
```
```bash
sudo exportfs -rav
sudo exportfs -v
```
Si a alguien `exportfs -v` le muestra una línea con `<world>`: dejó un espacio antes del paréntesis. Quitarlo y repetir `exportfs -rav`.

## Parte 4 — Firewall y comprobación
```bash
sudo firewall-cmd --permanent --add-service=nfs --add-service=rpc-bind --add-service=mountd
sudo firewall-cmd --permanent --zone=internal --add-service=nfs --add-service=rpc-bind --add-service=mountd
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --zone=internal --list-services
showmount -e localhost
```
Decir: "el firewall no filtra el tráfico de la propia máquina. Esto lo abrimos para un cliente real; hoy lo probamos contra nosotros mismos."

## Parte 5 — SELinux ya lo permite
```bash
getsebool nfs_export_all_ro nfs_export_all_rw use_nfs_home_dirs
```
