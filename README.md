# Bat-Egite

## Guía de instalación de Debian 12 usando debootstrap con Argon2id, Secure Boot, BTRFS y LUKS2.

Antes de nada necesitas saber o estar familiarizado con la terminal GNU/Linux y sus comandos.<br>
Sino es así aquí tienes una curso para ir empezando de la fundación GNU/Linux (https://training.linuxfoundation.org/training/introduction-to-linux/)

## Lo que necesitas:
<p>ISO en vivo de Debian Live (12.5.0 o posterior) - (https://www.debian.org/CD/live/)<br>
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
Primera ronda de borrado con /dev/random (Introduce datos aleatorios).
```bash 
root@debian:~# dd if=/dev/urandom of=/dev/sdb bs=4096 status=progress
```
Segunda ronda de borrado con /dev/zero (Introduce datos a zero o vacíos).
```bash 
root@debian:~# dd if=/dev/zero of=/dev/sdb bs=4096 status=progress
```
## Preparación del entorno de construcción
 Crearé una variable para el punto de montaje. Este es un paso importante ya que los comandos de esta guía llamarán a esta variable.
```bash 
root@debian:~# echo 'CB="/mnt"' >> ~/.bashrc
```
```bash 
root@debian:~# source ~/.bashrc
```
```bash 
root@debian:~# echo $CB
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
