GNU Hurd from the scratch

Advertencia!

El siguiente demo no es un sistema operativo completo, es decir no contiene ningun administrador de paquetes, no tiene conexion de red; solo tiene los paquetes necesarios para iniciar los servicios básicos y explorar un poco el sobre GNU Hurd, GNU Mach y GRUB.

Este demo no hubiera sido posible sin el trabajo y pasión de los hackers de Hurd, Debian, Guix, Nix y otros.

Este proyecto no provee scripts para construir la imagen de QEMU, ya que la idea es construir la imagen a través del administrador de paquetes Guix, pero provee los pasos para construir la imagen manualmente.


Objetivo:

La finalidad de construir este demo, es para entender el funcionamiento de conceptos como: compilacion cruzada, el funcionamiento de los paquetes binarios construidos a través del administrador de paquetes Guix.

Así como entender como funcionan e interactuan GNU Hurd y GNU Mach.


Pasos:

En Linux, crear la imagen necesaria para almacenar los binarios[1].

export LOOP=/dev/loop0
export MAPPER=/dev/mapper/loop0p1
export IMG=boot-guix.img
export IMG_SIZE=10240MB
sudo fallocate -l $IMG_SIZE $IMG
sudo losetup $LOOP $IMG 
sudo parted -a optimal -s $LOOP mklabel msdos 
sudo parted -a optimal -s $LOOP -- mkpart primary ext2 32k -1
sudo parted -s $LOOP -- set 1 boot on
sudo kpartx -a -v $LOOP
sudo mkfs.ext2 -o hurd -m 1 -v /dev/mapper/loop0p1 
mkdir rfs
sudo mount -t ext2 /dev/mapper/loop0p1 rfs
sudo umount ./rfs
sudo kpartx -d $LOOP
losetup -d $LOOP

Montar la imagen generada en Debian/Hurd, preparar el escenario para que funcione Guix y clonar el repositorio de Guix.

sudo qemu-system-i386 -enable-kvm -m 4G -hda debian-hurd.qcow2 -hdb boot-guix.img -curses

En Debian/Hurd montar el archivo,

sudo mkdir /gnu
sudo mount /dev/hd1s1 /gnu
sudo mkdir -p /gnu/store

Notas sobre como instalar Guix en Debian/Hurd,

https://lists.gnu.org/archive/html/guix-devel/2017-02/msg00865.html

Repositorio para descargar Guix:

Repositorio Oficial de Guix,
git://git.savannah.gnu.org/guix.git

Repositorio realizado con parches de Manolis y el equipo de Debian/Hurd,
https://github.com/methalo/guix/tree/guix-core-updates


Construir en Debian/Hurd los paquetes necesarios que se requieren para iniciar el GRUB, Mach y Hurd.

Para construir los binarios ejecutar,

cd guix
./pre-inst-env guix build hello --fallback --no-substitutes

Los binarios requeridos son los siguientes:

 *gnumach
 *coreutils
 *hurd
 *bash
 *xterm
 *file
 *shadow

Ejemplo para crear links y archivos necesarios[2],

En Debian/Hurd,

cd /gnu
mkdir sbin bin dev tmp root servers 
mkdir -p boot/grub
mkdir etc
touch etc/fstab
echo -n freedom > etc/hostname
touch servers/{exec,proc,password,default-pager} servers/crash-{dump-core,kill,suspend} 
cd servers
ln -s crash-dump-core crash
mkdir socket
touch socket/{1,2,16} 
cd socket
ln -s 1 local ; ln -s 2 inet ; ln -s 26 inet6

for i in "gnu/store/1sfij1j8ciyzm150n1j3qblchkjhn6f1-hurd-0.9/etc/"*                                            do                                                                                            
( cd etc ; ln -sv "$i" )                                                               
done

for i in "gnu/store/1sfij1j8ciyzm150n1j3qblchkjhn6f1-hurd-0.9/bin/"*
do                                                                                            
( cd bin ; ln -sv "$i" )                                                               
done
for i in "gnu/store/249g437mw539d5l1y8h5xnh2df1sb8s5-coreutils-8.27/bin/"*
do                                                                                            
( cd bin ; ln -sv "$i" )                                                               
done

cd bin
ln -s ../gnu/store/1sfij1j8ciyzm150n1j3qblchkjhn6f1-hurd-0.9/hurd hurd
ln -s ../gnu/store/n7ykcnx133wdfv8kbpznmjka6shfnzjl-bash-4.4.12/bin/bash sh
cd boot
ln -s ../gnu/store/xxrw52ksg02a8lp53scw3cj0n84f4yr6-gnumach-1.8/boot/gnumach gnumach
cp -rv gnu/store/18pn4bhgw8pbqqsxf4c1yywqz8c1799k-grub-2.02/lib/grub/i386-pc boot/grub

Los otros archivos necesarios se pueden tomar del repositorio.

Despues de crear todo lo necesario en Debian/Hurd desmontar la imagen, y montarla en Linux,

sudo mount -t ext2 /dev/mapper/loop0p1 rfs
sudo /gnu/store/w1hj0572wg91blbmsyk83zsnjl6j47ap-grub-2.02/sbin/grub-install --no-floppy --boot-directory rfs /dev/loop0


Depurar con subhurd[3],

jin@Hurd:~$ sudo boot /dev/hd1s1

Iniciar la imagen creada,

sudo qemu-system-i386 -enable-kvm -m 1G -hda boot-guix.img -curses


Referencias:
[1] https://github.com/flavioc/cross-hurd
[2] http://git.savannah.gnu.org/cgit/hydra-recipes.git/tree/hurd
[3] http://www.gnu.org/software/hurd/hurd/subhurd.html
# boot-hurd
