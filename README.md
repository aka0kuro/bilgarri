# Bat-Egite

## Guía de instalación de Debian 12 usando debootstrap con Argon2id, Secure Boot, BTRFS y LUKS2.

Antes de nada necesitas saber o estar familiarizado con la terminal GNU/Linux y sus comandos.<br>
Sino es así aquí tienes una curso para ir empezando de la fundación GNU/Linux (https://training.linuxfoundation.org/training/introduction-to-linux/)

## Lo que necesitas:
ISO en vivo de Debian Live (12.8.0 o posterior) - (https://www.debian.org/CD/live/)<br>
USB para gravar la ISO.<br>
Disco duro o SSD para la instalación.<br>
Dos ordenadores en la misma red.

## Empezemos
Empezamos por crear un USB con la iso de Debian Live, si no sabemos como crear el USB podemos usar BalenaEtcher (https://etcher.balena.io/) que es bastante intuitivo y fácil de usar o usar DD aquí una guía explicativa de como usar DD (https://www.cyberciti.biz/faq/creating-a-bootable-ubuntu-usb-stick-on-a-debian-linux/) que es mas complicado.<br>
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
Posterior mente introducimos IP a para ver que IP tenemos
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
root@debian:~# cryptsetup open /dev/sdx2 crypt
Enter passphrase for /dev/sdx2:
```
## Creamos y formateamos las particiones
Particionamos la particion EFI
```bash
root@debian:~# mkfs.vfat -nESP -F32 /dev/sdx1
mkfs.fat 4.2 (2021-01-31)
```
Particionamos la particion BTRFS
```bash
root@debian:~# mkfs.btrfs -L root /dev/mapper/crypt
btrfs-progs v6.2
See http://btrfs.wiki.kernel.org for more information.

NOTE: several default settings have changed in version 5.15, please make sure
      this does not affect your deployments:
      - DUP for metadata (-m dup)
      - enabled no-holes (-O no-holes)
      - enabled free-space-tree (-R free-space-tree)

Label:              root
UUID:               ca8743f2-c5f8-455c-b2e2-2031ecf183d9
Node size:          16384
Sector size:        4096
Filesystem size:    446.92GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP               1.00GiB
  System:           DUP               8.00MiB
SSD detected:       yes
Zoned device:       no
Incompat features:  extref, skinny-metadata, no-holes
Runtime features:   free-space-tree
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1   446.92GiB  /dev/mapper/crypt
```

## Creamos los subvolumes btrfs

primero creamos una variable de montaje
```bash
root@debian:~# mount_opt="rw,noatime,compress=zstd:1,ssd,discard=async,space_cache,space_cache=v2,commit=120"
```
Montamos las particiones
```bash
root@debian:~# mount -o $mount_opt /dev/mapper/crypt /mnt
root@debian:~# mkdir -p /mnt/boot/efi
root@debian:~# mount /dev/sdx1 /mnt/boot/efi/
```
Creamos los subvolumenes
```bash
root@debian:~# btrfs subvolume create /mnt/@
root@debian:~# btrfs subvolume create /mnt/@home
root@debian:~# btrfs subvolume create /mnt/@snapshots
root@debian:~# btrfs subvolume create /mnt/@swap
root@debian:~# btrfs subvolume create /mnt/@log
root@debian:~# btrfs subvolume create /mnt/@apt
root@debian:~# btrfs subvolume create /mnt/@crash
root@debian:~# btrfs subvolume create /mnt/@spool
```
## Montamos los subvolumenes

Creamos las carpetas para los subvolumenes
```bash
root@debian:~# mkdir -p /mnt/{swap,.snapshots,home,var/log,var/cache/apt,var/crash,var/spool}
```
Montamos los subvolumenes
```bash
root@debian:~# mount -o subvol=@snapshots,$mount_opt /dev/mapper/crypt /mnt/.snapshots
root@debian:~# mount -o subvol=@swap,$mount_opt /dev/mapper/crypt /mnt/swap
root@debian:~# mount -o subvol=@log,$mount_opt /dev/mapper/crypt /mnt/var/log
root@debian:~# mount -o subvol=@apt,$mount_opt /dev/mapper/crypt /mnt/var/cache/apt
root@debian:~# mount -o subvol=@crash,$mount_opt /dev/mapper/crypt /mnt/var/crash
root@debian:~# mount -o subvol=@spool,$mount_opt /dev/mapper/crypt /mnt/var/spool
```
Creamos el archivo de intercanbio<br>
calculadora de tamaño de swap (https://pickwicksoft.github.io/swapcalc/)
```bash
root@debian:~# chattr +C -R /mnt/swap
root@debian:~# btrfs filesystem mkswapfile --size 6G --uuid clear /mnt/swap/swapfile
root@debian:~# swapon /mnt/swap/swapfile
```
## Instalacion de Debian

Sistema base
```bash
root@debian:~# debootstrap --arch amd64 stable /mnt # Para Debian Stable
#root@debian:~# debootstrap --arch amd64 testing /mnt # Para Debian Testing
#root@debian:~# debootstrap --arch amd64 sid /mnt # Para Debian Sid
```
creamos el fstab
```bash
root@debian:~# genfstab -U /mnt >> /mnt/etc/fstab
```
acedemos al chroot
```bash
root@debian:~# arch-chroot /mnt /bin/bash
```
De primeras actualizamos los repositorios y los paquetes
```bash
root@debian:~# apt update && apt upgrade
```
Instalar locales y re-configurar tzdata
```bash
root@debian:/# apt install locales -y
```
Configurar tu idioma
```bash
root@debian:/# dpkg-reconfigure locales
```
Configurar zona horaria
```bash
root@debian:/# dpkg-reconfigure tzdata
```
Quitar el usuario root
```bash
root@debian:~# passwd -l root
```
Creamos el grupo wheel y el usuario
```bash
root@debian:~# addgroup wheel
```
```bash
root@debian:~# useradd -mG cdrom,floppy,sudo,audio,dip,video,plugdev,netdev -s /usr/bin/bash -c 'tusuario' tusuario
```
Contraseña para el tusuario
```bash
root@debian:/#  passwd tusuario
```
Ponemos el nombre al hostname y modificamos el archivos hosts
```bash
root@debian:~# HOSTNAME="tuhostname"
```
```bash
root@debian:~# echo "${HOSTNAME}" > /etc/hostname
```
```bash
root@debian:~# TAB="$(printf '\t')"
```
```bash 
root@debian:~# cat > /etc/hosts << EOF
127.0.0.1${TAB}localhost ${HOSTNAME}
::1${TAB}${TAB}localhost ip6-localhost ip6-loopback
fe00::0${TAB}${TAB}ip6-localnet
ff00::0${TAB}${TAB}ip6-mcastprefix
ff02::1${TAB}${TAB}ip6-allnodes
ff02::2${TAB}${TAB}ip6-allrouters
EOF
```
Añadir sources list stable, non-free, backports
```bash 
root@debian:~# apt install lsb-release
```
```bash 
root@debian:~# mv /etc/apt/sources.list /etc/apt/sources.list.old
```
```bash 
root@debian:~# CODENAME=$(lsb_release --codename --short)
```
```bash
root@debian:~# cat > /etc/apt/sources.list << EOF
deb http://deb.debian.org/debian ${CODENAME} main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian ${CODENAME} main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security/ ${CODENAME}-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security/ ${CODENAME}-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian ${CODENAME}-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian ${CODENAME}-updates main contrib non-free non-free-firmware

deb http://deb.debian.org/debian ${CODENAME}-backports main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian ${CODENAME}-backports main contrib non-free non-free-firmware
EOF
```
```bash
root@debian:~# apt update && apt upgrade
```
Instalamos las herramientas necesarias para initramfs y cryptsetup para poder crear /etc/crypttab para la partición raíz cifrada. 
```bash
root@debian:~# apt install btrfs-progs dosfstools cryptsetup-initramfs cryptsetup-suspend firmware-linux firmware-linux-nonfree sudo nano bash-completion command-not-found plocate console-setup fonts-terminus -y
```
Instalamos el kernel de backports
```bash 
root@debian:~# apt install linux-image-amd64/${CODENAME}-backports linux-headers-amd64/${CODENAME}-backports -y
```
Bloquemos grub y sus deribados ya que lo instalaremos desde el codigo fuente.
```bash
root@debian:/# apt-mark hold grub2 grub-pc grub-efi grub-efi-amd64
```
Instalamos los paquetes necesarios para compilar grub desde el código fuente
```bash 
root@debian:/# apt install --fix-missing shim-signed shim-helpers-amd64-signed sudo git curl libarchive-tools help2man python3 rsync texinfo ttf-bitstream-vera build-essential dosfstools efibootmgr uuid-runtime efivar mtools os-prober dmeventd libdevmapper-dev libdevmapper-event1.02.1 libdevmapper1.02.1 libfont-freetype-perl python3-freetype libghc-gi-freetype2-dev libghc-gi-freetype2-prof fuse2fs libconfuse2 gettext xorriso libisoburn-dev autogen gnulib libfreetype-dev pkg-config m4 libtool automake flex fuse3 libfuse3-dev gawk autoconf-archive rdfind fonts-dejavu lzma lzma-dev liblzma5 liblzma-dev liblz1 liblz-dev unifont acl libzfslinux-dev sbsigntool -y
```
creamos el directorio de claves y la clave para luks2
```bash 
root@debian:/# mkdir -vp /etc/keys
```
```bash 
root@debian:/# ( umask 0077 && dd if=/dev/urandom bs=1 count=128 of=/etc/keys/root.key conv=excl,fsync )
```
Asegúrese de configurar el propietario y el grupo como root y cambiar los permisos para que solo el propietario pueda leer, escribir y ejecutar.
```bash 
root@debian:/# chown -vR root:root /etc/keys
```
```bash 
root@debian:/# chmod -vR 600 /etc/keys
```
```bash 
root@debian:/# chattr +i /etc/keys/root.key
```
Agregamos la clave a LUKS2
```bash 
root@debian:/# cryptsetup --cipher aes-xts-plain64:sha512 -s 512 -h sha512 -i 10000 --pbkdf=argon2id luksAddKey /dev/sdx2 /etc/keys/root.key
```
Agregamos ahora luks2 particion a /etc/crypttab
```bash
root@debian:/# echo "crypt UUID=$(blkid -s UUID -o value /dev/sda2) /etc/keys/root.key luks,discard,key-slot=1" >> /etc/crypttab
```
Agregaré esta línea para que initramfs pueda encontrar la clave.
```bash 
root@debian:/# echo "KEYFILE_PATTERN=\"/etc/keys/*.key\"" >>/etc/cryptsetup-initramfs/conf-hook
```
Configuraré UMASK en un valor restrictivo para evitar la filtración de material clave. Luego me aseguraré de que se establezcan permisos restrictivos y que la clave esté disponible en initramfs.
```bash 
root@debian:/# echo UMASK=0077 >>/etc/initramfs-tools/initramfs.conf
```
Actualizamos el initrd
```bash 
root@debian:/# update-initramfs -v -u
```
Comprobamos los permisos y si la clave se encuentra
```bash 
root@debian:/# stat -L -c "%A %n" /initrd.img
```
```bash 
root@debian:/# lsinitramfs /initrd.img | grep "^cryptroot/keyfiles/"
```
Debe incluir gcc en la variable PATH o la compilación fallará para grub. Los otros son para que grub pueda encontrar bibliotecas adicionales que se clonan para la compilación. CFLAGS es algo que obtuve de un script que necesito encontrar nuevamente.
```bash 
root@debian:/# export PATH="$PATH:/bin/gcc:/sbin/gcc"
```
```bash 
root@debian:/# export GRUB_CONTRIB=./grub-extras
```
```bash 
root@debian:/# export GNULIB_SRCDIR=./gnulib
```
```bash 
root@debian:/# export CFLAGS=${CFLAGS/-fno-plt}
```
Una solución para que mawk dé errores durante el proceso de creación es cambiar el nombre de mawk en /usr/bin y luego señalar mawk a gawk:
```bash
root@debian:/# mv /usr/bin/mawk /usr/bin/mawk_bu
```
```bash
root@debian:/# ln -s /usr/bin/gawk /usr/bin/mawk
```
Crearé una carpeta de fuentes para la compilación de grub:
```bash
root@debian:/# mkdir -vp /sources && cd sources
```
Clonación de repositorios de git necesarios para el parche argon2 de Archlinux. 
```bash
root@debian:/sources# git clone https://git.savannah.gnu.org/git/grub.git
```
entramos en la carpeta grub
```bash
root@debian:/sources# cd grub
```
Descargamos los git necesarios
```bash
root@debian:/sources/grub# git clone https://git.savannah.nongnu.org/git/grub-extras.git
```
```bash
root@debian:/sources/grub#  git clone https://aur.archlinux.org/grub-improved-luks2-git.git
```
```bash
root@debian:/sources/grub#  git clone https://git.savannah.gnu.org/git/gnulib.git
```
Esta parte está copiada de grub-improved-luks2-git/PKGBUILD. Parcha grub y lo compila e instala.
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/add-GRUB_COLOR_variables.patch
patching file util/grub-mkconfig.in
Hunk #1 succeeded at 250 (offset 4 lines).
patching file util/grub.d/00_header.in
```
Parche grub-mkconfig para detectar imágenes initramfs de Arch Linux.
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/detect-archlinux-initramfs.patch
patching file util/grub.d/10_linux.in
Hunk #1 succeeded at 95 (offset 2 lines).
Hunk #2 succeeded at 212 (offset 12 lines).
Hunk #3 succeeded at 301 with fuzz 1 (offset 14 lines).
```
Argón2:
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_1.patch
patching file grub-core/kern/dl.c
Hunk #1 succeeded at 470 (offset 3 lines).
patching file util/grub-module-verifierXX.c
Hunk #1 succeeded at 236 with fuzz 1 (offset 79 lines).
```
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_2.patch
patching file include/grub/types.h
Hunk #1 succeeded at 156 (offset 3 lines).
Hunk #2 succeeded at 178 (offset 3 lines).
```
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_3.patch
patching file docs/grub-dev.texi
Hunk #1 succeeded at 503 (offset 13 lines).
patching file grub-core/Makefile.core.def
Hunk #1 succeeded at 1219 (offset 45 lines).
patching file grub-core/lib/argon2/LICENSE
patching file grub-core/lib/argon2/argon2.c
patching file grub-core/lib/argon2/argon2.h
patching file grub-core/lib/argon2/blake2/blake2-impl.h
patching file grub-core/lib/argon2/blake2/blake2.h
patching file grub-core/lib/argon2/blake2/blake2b.c
patching file grub-core/lib/argon2/blake2/blamka-round-ref.h
patching file grub-core/lib/argon2/core.c
patching file grub-core/lib/argon2/core.h
patching file grub-core/lib/argon2/ref.c
```
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_4.patch
patching file grub-core/disk/luks2.c
Hunk #1 succeeded at 38 (offset -2 lines).
Hunk #2 succeeded at 91 (offset -2 lines).
Hunk #3 succeeded at 161 (offset -2 lines).
Hunk #4 succeeded at 461 (offset 14 lines).
```
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/argon_5.patch
patching file Makefile.util.def
patching file grub-core/Makefile.core.def
Hunk #1 succeeded at 1242 (offset 45 lines).
patching file grub-core/disk/luks2.c
Hunk #2 succeeded at 463 (offset 14 lines).
```
Haga que grub-install funcione con luks2:
```bash
root@debian:/sources/grub#  patch -Np1 -i ./grub-improved-luks2-git/grub-install_luks2.patch
patching file util/grub-install.c
Hunk #1 succeeded at 448 (offset 2 lines).
```
Corrija la ubicación de DejaVuSans.ttf para que grub-mkfont pueda crear archivos *.pf2 para el tema starfield:
```bash
root@debian:/sources/grub#  sed 's|/usr/share/fonts/dejavu|/usr/share/fonts/dejavu /usr/share/fonts/truetype/dejavu|g' -i "configure.ac"
```
Modifique el comportamiento de grub-mkconfig para silenciar las advertencias FS#36275
```bash
root@debian:/sources/grub#  sed 's| ro | rw |g' -i "util/grub.d/10_linux.in"
```
Modifique el comportamiento de grub-mkconfig para que las entradas generadas automáticamente lean 'Arch Linux' FS#33393
```bash
root@debian:/sources/grub#  sed 's|GNU/Linux|Linux|' -i "util/grub.d/10_linux.in"
```
Elimine el módulo lua de grub-extras ya que es incompatible con los cambios en grub_file_open. http://git.savannah.gnu.org/cgit/grub.git/commit/?id=ca0a4f689a02c2c5a5e385f874aaaa38e151564e
```bash
root@debian:/sources/grub#  rm -rf ./grub-extras/lua
```
Arrancaré grub2 y luego lo compilaré:
```bash
root@debian:/sources/grub#  ./bootstrap
```
```bash
root@debian:/sources/grub# mkdir ./build_x86_64-efi
```
```bash
root@debian:/sources/grub# cd ./build_x86_64-efi
```
Enpezamos la compilacion
```bash 
root@debian:/sources/grub/build_x86_64-efi#  ../configure --with-platform=efi --target=x86_64 --prefix="/usr" --sbindir="/usr/bin" --sysconfdir="/etc" --enable-boot-time --enable-cache-stats --enable-device-mapper --enable-grub-mkfont --enable-grub-mount --enable-mm-debug --disable-silent-rules --disable-werror  CPPFLAGS="$CPPFLAGS -O2" --enable-stack-protector --enable-liblzma
```
Deberías obtener algo como esto:
```code
*******************************************************
GRUB2 will be compiled with following components:
Platform: x86_64-efi
With devmapper support: Yes
With memory debugging: Yes
With disk cache statistics: Yes
With boot time statistics: Yes
efiemu runtime: No (not available on efi)
grub-mkfont: Yes
grub-mount: Yes
starfield theme: Yes
With DejaVuSans font from /usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
With libzfs support: Yes
Build-time grub-mkfont: Yes
With unifont from /usr/share/fonts/X11/misc/unifont.pcf.gz
With liblzma from -llzma (support for XZ-compressed mips images)
With stack smashing protector: Yes
*******************************************************
```
Si todo parece bien, instalamos grub2:
```bash 
root@debian:/sources/grub/build_x86_64-efi#  make DESTDIR=/ bashcompletiondir=/usr/share/bash-completion/completions install
```        
```bash 
root@debian:/sources/grub/build_x86_64-efi#  install -D -m0644 ../grub-improved-luks2-git/grub.default /etc/default/grub
```
```bash 
root@debian:/sources/grub/build_x86_64-efi#  cd ../../..
```
Cambie el título del menú por el tuyo.
```bash 
root@debian:/# sed -i 's|GRUB_DISTRIBUTOR="Arch"|GRUB_DISTRIBUTOR="Kaisen"|g' /etc/default/grub
```
Habilite cryptodisk para darle a grub la capacidad de desbloquear la partición cifrada en el momento del arranque para acceder a initramfs en el directorio /boot
```bash 
root@debian:/# sed -i 's|#GRUB_ENABLE_CRYPTODISK=y|GRUB_ENABLE_CRYPTODISK=y|g' /etc/default/grub
```
Crearé un nuevo archivo de configuración de grub:
```bash
root@debian:/# grub-mkconfig -o /boot/grub/grub.cfg
```
Crearé un sbat.csv:
```bash 
root@debian:/# cat > /usr/share/grub/sbat.csv << EOF
sbat,1,SBAT Version,sbat,1,https://github.com/rhboot/shim/blob/main/SBAT.md
grub,4,Free Software Foundation,grub,2.12,https://www.gnu.org/software/grub/
grub.debian,5,Debian,grub2,2.12-2kaisen,https://tracker.debian.org/pkg/grub2
grub.debian13,1,Debian,grub2,2.12-2kaisen,https://tracker.debian.org/pkg/grub2
grub.peimage,2,Canonical,grub2,2.12-2kaisen,https://salsa.debian.org/grub-team/grub/-/blob/master/debian/patches/secure-boot/efi-use-peimage-shim.patch
EOF
```
Es hora de instalar el grub efi:
```bash 
root@debian:/# grub-install --target=x86_64-efi --efi-directory=/boot/efi --boot-directory=/boot --modules="bli argon2 all_video boot btrfs cat chain configfile echo efifwsetup efinet ext2 fat font gettext gfxmenu gfxterm gfxterm_background gzio halt help hfsplus iso9660 jpeg keystatus loadenv loopback linux ls lsefi lsefimmap lsefisystab lssal memdisk minicmd normal ntfs part_apple part_msdos part_gpt password_pbkdf2 png probe reboot regexp search search_fs_uuid search_fs_file search_label serial sleep smbios squash4 test tpm true video xfs zfs zfscrypt zfsinfo cpuid play cryptodisk gcry_arcfour gcry_blowfish gcry_camellia gcry_cast5 gcry_crc gcry_des gcry_dsa gcry_idea gcry_md4 gcry_md5 gcry_rfc2268 gcry_rijndael gcry_rmd160 gcry_rsa gcry_seed gcry_serpent gcry_sha1 gcry_sha256 gcry_sha512 gcry_tiger gcry_twofish gcry_whirlpool luks luks2 lvm mdraid09 mdraid1x raid5rec raid6rec" --sbat /usr/share/grub/sbat.csv /dev/sdx
```
Pasaré por encima del administrador de shim y mok firmado, así como del grubx64.efi que acabamos de crear con grub-install. 
```bash 
root@debian:/# mkdir -vp /boot/efi/EFI/BOOT
```
```bash 
root@debian:/# cp /boot/efi/EFI/debian/grubx64.efi /boot/efi/EFI/BOOT/
```
```bash 
root@debian:/# cp /usr/lib/shim/shimx64.efi.signed /boot/efi/EFI/BOOT/bootx64.efi
```
```bash 
root@debian:/# cp /usr/lib/shim/mmx64.efi.signed /boot/efi/EFI/BOOT/mmx64.efi
``` 
Crearé la clave del propietario de la máquina junto con el certificado necesario al registrarla en el administrador MOK en el primer arranque. También estableceré permisos restrictivos:
```bash
root@debian:/# mkdir -vp /etc/keys/MOK
```
```bash 
root@debian:/# (umask 0077 && openssl req -newkey rsa:2048 -nodes -keyout /etc/keys/MOK/MOK.key -new -x509 -sha256 -days 3650 -subj "/CN=el nombre que quieras/" -out /etc/keys/MOK/MOK.crt)
```
```bash
root@debian:/# (umask 0077 && openssl x509 -outform DER -in /etc/keys/MOK/MOK.crt -out /etc/keys/MOK/tunombre-MOK.cer)
```
```bash
root@debian:/# mkdir -vp /efi/EFI/certs
```
```bash
root@debian:/# cp /etc/keys/MOK/tunombre-MOK.cer /boot/efi/EFI/certs/
```
```bash 
root@debian:/# chmod -vR 600 /etc/keys
```
```bash 
root@debian:/# chattr +i /etc/keys/MOK/MOK.key
```
```bash 
root@debian:/# chattr +i /etc/keys/MOK/tunombre-MOK.cer
```
```bash
root@debian:/# chattr +i /etc/keys/MOK/MOK.crt
```
Ahora comprobaré las firmas y firmaré vmlinuz y grubx64.
```bash 
root@debian:/# sbverify --list /boot/$(ls /boot | grep vmlinuz)
```
Si tienes ya alguna firma para borrarla.
```bash
root@debian:/# sbattach --signum 1 --remove /boot/$(ls /boot | grep vmlinuz)
```
```bash 
root@debian:/# sbsign --key /etc/keys/MOK/MOK.key --cert /etc/keys/MOK/MOK.crt --output /boot/$(ls /boot | grep vmlinuz) /boot/$(ls /boot | grep vmlinuz)
```
```bash 
root@debian:/# sbverify --list /boot/$(ls /boot | grep vmlinuz)
```
```bash 
root@debian:/# sbverify --list /boot/efi/EFI/BOOT/grubx64.efi
```
```bash 
root@debian:/# sbattach --signum 1 --remove /boot/efi/EFI/BOOT/grubx64.efi  
```
```bash 
root@debian:/# sbsign --key /etc/keys/MOK/MOK.key --cert /etc/keys/MOK/MOK.crt --output /boot/efi/EFI/BOOT/grubx64.efi /boot/efi/EFI/BOOT/grubx64.efi
```
```bash 
root@debian:/# sbverify --list /boot/efi/EFI/BOOT/grubx64.efi
``` 
```bash 
root@debian:/# efibootmgr -c -d /dev/sdb -p 1 -L  "tuNombre" -l '\EFI\BOOT\bootx64.efi'
```
Por si quieres borrar alguna entrada donde la x es el numero o letra del final.
```bash 
root@debian:/# sudo efibootmgr --delete-bootnum --bootnum  x
```
