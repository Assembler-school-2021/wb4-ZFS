# wb4-ZFS

Aprovisionar servidor
Aprovisionaremos un debian 10 y a parte le conectaremos 4 volúmenes de 20gb cada uno. Dejando los 4 primeros volumenes de 20gb y el último de 10. Instalareemos una debian minimal en el 5o volumen, o sea sde

![image](https://user-images.githubusercontent.com/65896169/126286322-f2573f01-ec24-41f8-aa84-9efa80712684.png)

Particionar los discos duros
Instalar los siguientes paquetes:

```
apt install gdisk rsync
```
A continuación particionaremos los 4 discos adecuadamente.

```
gdisk /dev/sda
gdisk /dev/sdb
gdisk /dev/sdc
gdisk /dev/sdd
```
```
# gdisk /dev/sdb
GPT fdisk (gdisk) version 0.8.6

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.
Command (? for help): **p**
Disk /dev/sda: 488397168 sectors, 232.9 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): 948B55B3-300A-4610-A9D0-022FD36BD186
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 488397134
Partitions will be aligned on 2048-sector boundaries
Total free space is 488397101 sectors (232.9 GiB)


Number  Start (sector)    End (sector)  Size       Code  Name


Command (? for help): **n**
Partition number (1-128, default 1): **9**
First sector (34-488397134, default = 2048) or {+-}size{KMGTP}: **-512M**
Information: Moved requested sector from 487348558 to 487348224 in
order to align on 2048-sector boundaries.
Use 'l' on the experts' menu to adjust alignment
Last sector (487348224-488397134, default = 488397134) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): **ef02**
Changed type of partition to 'BIOS boot partition'

Command (? for help): **n**
Partition number (1-128, default 1): 
First sector (34-487348223, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-487348223, default = 487348223) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): **bf01**
Changed type of partition to 'Solaris /usr & Mac ZFS'

Command (? for help): **x**

Expert command (? for help): **a**
Partition number (1-9): **9**
Known attributes are:
0: system partition
1: hide from EFI
2: legacy BIOS bootable
60: read-only
62: hidden
63: do not automount

Attribute value is 0000000000000000. Set fields are:
  No fields set

Toggle which attribute field (0-63, 64 or <Enter> to exit): **2**
Have enabled the 'legacy BIOS bootable' attribute.
Attribute value is 0000000000000004. Set fields are:
2 (legacy BIOS bootable)

Toggle which attribute field (0-63, 64 or <Enter> to exit): 

Expert command (? for help): **m**

Command (? for help): **w**
```

Repetir el proceso con los otros 3 discos.

Instalar ZFS y configurar:
En primer lugar, añadir el siguiente repositorio:

```
echo "deb http://deb.debian.org/debian buster-backports main" >> /etc/apt/sources.list
sed -i 's/main/main contrib/g' /etc/apt/sources.list
apt update
```
Instalar los linux headers:

```
apt install linux-headers-`uname -r`
```
A continuación instalaremos ZFS.

```
apt install -t buster-backports dkms spl-dkms
apt install -t buster-backports zfs-dkms zfsutils-linux
apt install zfs-initramfs
echo REMAKE_INITRD=yes > /etc/dkms/zfs.conf
```
Y crearemos el pool zfs, configurando una quota máxima.

```
zpool create -d -o feature@async_destroy=enabled -o feature@empty_bpobj=enabled -o feature@lz4_compress=enabled -o ashift=12 -O compression=lz4 rpool raidz2 /dev/sd[a-d]1 -f
zfs create rpool/root
zfs set quota=10gb rpool/root
```
Copia el sistema actual al nuevo filesystem.
```
cd /rpool/root/
rsync -a --one-file-system / /rpool/root/
```
Preparamos chroot y a continuación lo ejecutamos:
```
mount --bind /dev dev
mount --bind /proc proc
mount --bind /sys sys
mount --bind /run run
chroot .

```
Configura el grub -> /etc/default/grub :
```
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="nomodeset consoleblank=0"
GRUB_CMDLINE_LINUX="boot=zfs root=ZFS=rpool/root"
GRUB_PRELOAD_MODULES="part_gpt zfs"
```

GRUB_CMDLINE_LINUX_DEFAULT="nomodeset consoleblank=0"
Commenta la antigua entrada root de /etc/fstab:
```
#UUID=8629f208-0ff4-4f52-89d8-20ca45c87631 / ext4 defaults 0 0
Acaba la instalación del grub y zfs :
export ZPOOL_VDEV_NAME_PATH=YES
update-initramfs -c -k all
update-grub
grub-install /dev/sda
grub-install /dev/sdb
grub-install /dev/sdc
grub-install /dev/sdd
zfs set mountpoint=/ rpool/root
exit
```
Apagar la máquina.
```
shutdown now
```
Si todo ha funcionado bien, desconecta el volúmen del sistema operativo y tiene que arrancar el vps de el root del zfs.

Ejercicios:

> Según la commandline empleada, que nivel de zraid hemos usado?

> Que toleráncia a fallidas de discos tenemos?

> Cómo nos aseguraremos de comprobar los datos de la cabina cada 7 días?

> Cómo podemos comprobar el estado del pool?

> Crea un pool para un cliente de 5gb.

> Crea otro pool para otro cliente de 20g.

> Cuanto espacio libre nos queda ahora?
