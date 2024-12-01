# Bat-Egite

## Guía de instalación de Debian 12 usando debootstrap con Argon2id, Secure Boot, BTRFS y LUKS2.

Antes de nada necesitas saber o estar familiarizado con la terminal GNU/Linux y sus comandos.<br>
Sino es así aquí tienes una curso para ir empezando de la fundación GNU/Linux (https://training.linuxfoundation.org/training/introduction-to-linux/)

## Lo que necesitas:
<p>ISO en vivo de Debian Live (12.8.0 o posterior) - (https://www.debian.org/CD/live/)<br>
USB para gravar la ISO.<br>
Disco duro o SSD para la instalación.<br>
Dos ordenadores en la misma red.</p>

## Empezemos
<p>Empezamos por crear un USB con la iso de Debian Live, si no sabemos como crear el USB podemos usar BalenaEtcher (https://etcher.balena.io/) que es bastante intuitivo y fácil de usar o usar DD aquí una guía explicativa de como usar DD (https://www.cyberciti.biz/faq/creating-a-bootable-ubuntu-usb-stick-on-a-debian-linux/) que es mas complicado.<br>
Cuando este creado el USB con Debian Live lo arrancamos, en el ordenador <strong>server</strong> como lo llamaremos de ahora en adelante.<br>

## Arrancamos
Arrancamos Debian Live en el ordenador <strong>server</strong>.<br>
Cuando este cargado el escritorio del <strong>server</strong> abrimos una terminal e instalamos openssh-server.<br><br>


```bash
user@debian:~$ sudo apt install openssh-server -y
```
Después de de la instalación arrancaremos el demonio de SSH<br><br>
```bash
user@debian:~$ sudo systemctl start ssh
```
Posterior mente introducimos IP a para ver que IP tenemos</p>
```bash
user@debian:~$ ip a
```
## Maquina cliente

En la maquina cliente que a partir de ahora se llamara <strong>cliente</strong> nos conectaremos al <strong>server</strong> todo ello por SSH.
```bash
tusuario@tumaquina:~$ ssh user@IP_del_server
```
Con contraseña <strong>live</strong><br><br>
Yo personalmente borro el SSD (en mi caso) dos veces.<br>
Entramos en modo superusuario.
```bash
user@debian:~$ sudo su -
```
Preparate un cafe o abrete una cerveza que esto tardara bastante.<br>
Primera ronda de borrado con /dev/random (Introduce datos aleatorios).<br>
Donde <strong>x</strong> es tu disco duro.
```bash
root@debian:~# dd if=/dev/urandom of=/dev/sdx bs=4096 status=progress
```
Segunda ronda de borrado con /dev/zero (Introduce datos a zero o vacíos).
```bash
root@debian:~# dd if=/dev/zero of=/dev/sdx bs=4096 status=progress
```
## Preparación del entorno de construcción
 Crearé una variable para el punto de montaje. Este es un paso importante ya que los comandos de esta guía llamarán a esta variable.
```bash
root@debian:~# echo 'MN="/mnt"' >> ~/.bashrc
```
```bash
root@debian:~# source ~/.bashrc
```
```bash
root@debian:~# echo $MN
/mnt
```
## Instalación de dependencias
Colocamos los repositorios non-free y nonfree-firmware.
```bash
root@debian:~# sed 's|deb http://deb.debian.org/debian/ bookworm main non-free-firmware|deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware|g' -i "/etc/apt/sources.list"
```
```bash
root@debian:~# apt update && apt upgrade
```
Instalamos el paquete debootstrap y arch-install-script. Por que el arch-install-script por que generar el fstab es mucho mas sencillo y no hay que introducirlo a mano.
```bash
root@debian:~# apt install arch-install-scripts debootstrap -y
```
Usaremos <strong>gdisk</strong> para particionar.
```bash
root@debian:~# gdisk /dev/sdx
```
```code
  GPT fdisk (gdisk) version 1.0.9

  Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

  Creating new GPT entries in memory.

  Command (? for help): o
  This option deletes all partitions and creates a new protective MBR.
  Proceed? (Y/N): y

  Command (? for help): n
  Partition number (1-128, default 1):  
  First sector (34-937703054, default = 2048) or {+-}size{KMGTP}:
  Last sector (2048-937703054, default = 937701375) or {+-}size{KMGTP}: +200M
  Current type is 8300 (Linux filesystem)
  Hex code or GUID (L to show codes, Enter = 8300): ef00
  Changed type of partition to 'EFI system partition'

  Command (? for help): n
  Partition number (2-128, default 2):
  First sector (34-937703054, default = 411648) or {+-}size{KMGTP}:
  Last sector (411648-937703054, default = 937701375) or {+-}size{KMGTP}:
  Current type is 8300 (Linux filesystem)
  Hex code or GUID (L to show codes, Enter = 8300):
  Changed type of partition to 'Linux filesystem'

  Command (? for help): p
  Disk /dev/sda: 937703088 sectors, 447.1 GiB
  Model: KINGSTON SA400S3
  Sector size (logical/physical): 512/512 bytes
  Disk identifier (GUID): 2F9E0BEF-818D-416F-9929-A7F1C476B67F
  Partition table holds up to 128 entries
  Main partition table begins at sector 2 and ends at sector 33
  First usable sector is 34, last usable sector is 937703054
  Partitions will be aligned on 2048-sector boundaries
  Total free space is 3693 sectors (1.8 MiB)

  Number  Start (sector)    End (sector)  Size       Code  Name
     1            2048          411647   200.0 MiB   EF00  EFI system partition
     2          411648       937701375   446.9 GiB   8300  Linux filesystem

  Command (? for help): w

  Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
  PARTITIONS!!

  Do you want to proceed? (Y/N): y
  OK; writing new GUID partition table (GPT) to /dev/sda.
  The operation has completed successfully.
```
## Creación de la partición Luks2
Utilizamos argon2id con luks2 para el encriptado.<br>
```bash
root@debian:~# cryptsetup luksFormat --pbkdf=argon2id --use-urandom -s 512 -h sha512 -i 10000 /dev/sdx2
```
```code
WARNING!
========
This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sda2:
Verify passphrase:
```
Abrimos la particion encriptada.
```bash
root@debian:~# cryptsetup open /dev/sda2 crypt
Enter passphrase for /dev/sda2: 
```
