# How to encrypt the Pinenote 2024

This is a guide for the experienced Linux administrator. I won't go into every detail, but the adventurous might be able to fill the gaps with their favorite llm.

It is possible to boot into encrypted root and data partitions on the pinetab 2024. A key feature of the stock installation is that it offers two different OS partition. To keep sanity during the process I recommend to modify OS2 only and always keep OS1 bootable. Also using the UART adapter is recommended to get all info on the console.  

## Partitioning

The first step is to make some room for an unencrypted boot partiton. This has to be number 6 if you want to avoid changes to u-boot. So, delete old OS2 and make a small boot and a root partition to fill the space.

My parition table after the modifications looks like:  

Number  Start   End     Size    File system  Name       Flags  
 1      8389kB  75.5MB  67.1MB               uboot  
 2      75.5MB  77.6MB  2097kB               waveform  
 3      77.6MB  78.6MB  1049kB               uboot_env  
 4      78.6MB  146MB   67.1MB               logo  
 5      146MB   15.9GB  15.7GB  ext4         os1        legacy_boot  
 6      15.9GB  16.4GB  525MB   ext4         boot  
 8      16.4GB  31.6GB  15.2GB               root  
 7      31.6GB  124GB   92.1GB               data   
 
## Preparing the boot partition
 
     mkfs.ext4 /dev/mmcblk0p6
     mkdir /mnt/tmp
     mount /dev/mmcblk0p6 /mnt/tmp
     cp -a /boot /mnt/tmp/
     cp -a  /usr/lib/linux-image-6.12.11-pinenote-202501281646-00249-g211ba27556cc/ /mnt/tmp/boot     

Then we need to update two lines in each option in boot/extlinux/extlinux.conf. The path for the dtb has to be within our new boot partition. And we need to add the cryptdevice option and change the root device to /dev/mapper/root. In my case these two lines look like:

    fdt /boot/linux-image-6.12.11-pinenote-202501281646-00249-g211ba27556cc/rockchip/rk3566-pinenote-v1.2.dtb  
    append cryptdevice=/dev/mmcblk0p8:root root=/dev/mapper/root ignore_loglevel rw rootwait earlycon console=tty0 console=ttyS2,1500000n8 fw_devlink=off quiet loglevel=3 systemd.show_status=auto rd.udev.log_level=3 splash plymouth.ignore-serial-consoles vt.global_cursor_default=0  

## Prepairing data

That is done in the usual way using cryptsetup (also see below), nothing special here. Just remember to remove /dev/mmcblk0p8 from /etc/fstab in OS1. Just copy everything you need there directly to the dir /home on the OS1 partition. 

## Preparing root

    umount /mnt/tmp
    cryptsetup -c aes-xts-plain:sha256 -y -s 512 luksFormat /dev/mmcblk0p8
    cryptsetup luksOpen /dev/mmcblk0p8 root
    mkfs.ext4 /dev/mapper/root
    mount /dev/mapper/root /mnt/tmp
    cp -ax / /mnt/tmp
    rm -rf /mnt/tmp/boot
    mkdir /mnt/tmp/boot2
    mount /dev/mmcblk0p6 /mnt/tmp/boot2
    cd /mnt/tmp
    ln -s boot2 boot

The strange arrangement with boot2 is necessary, because u-boot looks for the files in the boot subdirectory of the partition. And the system later also wants to have them in /boot. The sane solution there would be to change uboot.env, which I want to avoid to reduce the complexity of the solution. 

Now I use the following script to chroot into our new system:

    mount --bind /dev $1/dev
	mount --bind /dev/pts $1/dev/pts
	mount -t proc proc $1/proc
	mount -t sysfs sysfs $1/sys
	cp -L /etc/resolv.conf $1/etc/resolv.conf
	chroot $1
	

and call it with ./scriptname.sh /mnt/tmp

To mount /home you need to use

	cryptdisks_start home
    mount -a

(after updating fstab and crypttab, see below)


You need to have the follwing packages installed:  

    cryptsetup
    cryptsetup-bin
	cryptsetup-initramfs
    libcryptsetup12:arm64
    unl0kr
    cryptmount
    systemd-cryptsetup 

/etc/fstab should look something like 

    /dev/mapper/root / ext4 defaults 0 1
    /dev/mmcblk0p6 /boot2 ext4 defaults 0 0 
    /dev/disk/by-partlabel/uboot_env /uboot_config             vfat          defaults,noauto              0      0
    /dev/mapper/home /home              ext4   defaults	0	0

Make a passphrase for home partition  

    dd if=/dev/random of=~/secret bs=1k count=4
    cryptsetup luksAddKey  /dev/mmcblk0p7 /root/secret
    

/etc/cryprtab should look like:

    root /dev/mmcblk0p8 none luks,initramfs,keyscript=/usr/share/initramfs-tools/  scripts/unl0kr-keyscript
    home /dev/mmcblk0p7 /root/secret luks


To enable touch we need to add cyttsp5 to the modules loaded in initramfs:

    echo cyttsp5 >> /etc/initramfs-tools/modules
 
There seems to be a race condition loading that module, therefore we need a file with the content

    install cyttsp5 /bin/sh -c 'sleep 2; /sbin/modprobe --ignore-install cyttsp5'

in a file like delay_cyttsp5.conf in /etc/modprobe.d

Now the last thing left is to mount all partitions (in particular home) and  

    update-initramfs -u  
    shutdown -r
 

## Unwanted side effects

Booting OS1 now take something like 30 seconds to load the kernel. 


