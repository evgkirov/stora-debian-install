# How to install Debian Wheezy on Netgear Stora

This is how I installed Debian Wheezy on Netgear Stora.


## Credits

The following how-to is mostly a copypaste from various sources:

* http://forum.doozan.com/read.php?2,5986,page=1
* http://www.openstora.com/wiki/index.php?title=How_to_install_Debian_Linux_on_NETGEAR_Stora
* https://github.com/johnou/debian_stora_kernel


## How to install

You will need:

* Netgear Stora
* USB stick (the system will be running from it — so it should be fast enough)
* Serial console access

First prepare your USB stick. Create an ext2 partition and label it as `root`.

Download [rootfs image](https://github.com/kirov/stora-debian-install/releases/download/v20140114/stora-debian-rootfs_20140114.tar.gz) and unpack it to your flash drive.

Connect [serial cable](http://www.openstora.com/wiki/index.php?title=Root_Access_Via_Serial_Console) to your Stora. Power on the unit, press any key when you see `Hit any key to stop autoboot` and enter:

    setenv mainlineLinux yes
    setenv arcNumber 2743
    setenv bootcmd_usb 'usb reset; ext2load usb 0 0x200000 /boot/uImage; ext2load usb 0 0x800000 /boot/uInitrd'
    setenv bootcmd 'setenv bootargs $(console) root=LABEL=root rootdelay=8; run bootcmd_usb; bootm 0x200000 0x800000'
    saveenv
    reset

Stora will reboot and Debian will start. Default root password: `111222`.


## How to build your own rootfs

You can build your own rootfs if you want. You will need Debian or Ubuntu machine for this.

First, you need to install debootstrap, create an img-file for the rootfs (1GB should be fine) and mount it

    sudo -s
    apt-get install debootstrap
    dd if=/dev/zero of=debian_rootfs.img bs=1M count=1024
    mkfs.ext3 -F debian_rootfs.img
    mkdir /media/debian
    mount debian_rootfs.img /media/debian -o loop

Now let's run the first stage of debootstrap and get Debian Wheezy armel port

    debootstrap --verbose --arch=armel --variant=minbase --foreign wheezy /media/debian http://cdn.debian.net/debian/

The output should look like [this](http://pastebin.com/C2iNHysE). With the aid of Binfmt_misc we are able to chroot into armel rootfs on a x86 machine.

    apt-get install binfmt-support qemu-user-static
    modprobe binfmt_misc
    cp /usr/bin/qemu-arm-static /media/debian/usr/bin
    mkdir /media/debian/dev/pts
    mount devpts /media/debian/dev/pts -t devpts
    mount -t proc proc /media/debian/proc
    chroot /media/debian/

If you see something like "I have no name!@hostname:/#", you're inside. Let's proceed with the second stage of debootstrap

    /debootstrap/debootstrap --second-stage

The output should look like [this](http://pastebin.com/xiXvdNWq). At the end you should see

    I: Base system installed successfully.

Now we can start installing necessary packages and configuring our rootfs. But before we must add the package sources.

    cat <<END > /etc/apt/sources.list
    deb http://cdn.debian.net/debian wheezy main
    deb-src http://cdn.debian.net/debian wheezy main
    deb http://cdn.debian.net/debian wheezy-updates main
    deb-src http://cdn.debian.net/debian wheezy-updates main
    deb http://security.debian.org/ wheezy/updates main
    deb-src http://security.debian.org/ wheezy/updates main
    END
    apt-get update

At this point it's a good idea to configure the language settings

    exit
    mount devpts /media/debian/dev/pts -t devpts
    mount -t proc proc /media/debian/proc
    chroot /media/debian/
    export LANG=C
    apt-get install apt-utils
    apt-get install dialog
    apt-get install locales
    cat <<END > /etc/apt/apt.conf
    APT::Install-Recommends "0";
    APT::Install-Suggests "0";
    END
    dpkg-reconfigure locales

If you need English only, just select en_US.UTF-8 in both dialogs and run

    export LC_ALL=en_US.UTF-8

after that. Don't select encodings other than UTF-8. For other languages, select en_US.UTF-8 and the appropriate locale for your language (xx_YY.UTF-8). For example, if you speak French, select en_US.UTF-8 and fr_FR.UTF-8 in the first dialog and choose fr_FR.UTF-8 in the second dialog. After that run

    export LC_ALL=fr_FR.UTF-8

This way Debian will speak your mother tongue. The output shoud look like [this](http://pastebin.com/hTR01wUh).

It's time to install the kernel. You can [build your own kernel](https://github.com/kirov/stora-debian-kernel) if you want.

    cd /root
    rm -f /etc/blkid.tab
    ln -s /dev/null /etc/blkid.tab
    rm -f /etc/mtab
    ln -s /proc/mounts /etc/mtab
    apt-get install wget ca-certificates initramfs-tools uboot-mkimage
    wget https://github.com/kirov/stora-debian-kernel/releases/download/v3.10.26/linux-image-3.10.26-stora_1_armel.deb
    wget https://github.com/kirov/stora-debian-kernel/releases/download/v3.10.26/linux-headers-3.10.26-stora_1_armel.deb
    dpkg -i linux-image-3.10.26-stora_1_armel.deb
    dpkg -i linux-headers-3.10.26-stora_1_armel.deb

We need to generate uBoot and uInitrd

    mkimage -A arm -O linux -T kernel  -C none -a 0x00008000 -e 0x00008000 -n Linux-3.10.26-stora -d /boot/vmlinuz-3.10.26-stora /boot/uImage
    mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs-3.10.26-stora -d /boot/initrd.img-3.10.26-stora /boot/uInitrd

Install more packages

    apt-get install kmod dhcp3-client udev netbase ifupdown iproute openssh-server iputils-ping net-tools ntpdate vim nano

Now we are going to reproduce some basic configuration steps. Starting with network configuration

    cat <<END > /etc/network/interfaces
    auto lo eth0
    iface lo inet loopback
    iface eth0 inet dhcp
    END

Setup machine's name (in this case it will be "stora")

    echo stora > /etc/hostname

Activate remote console and disable local consoles

    echo 'T0:2345:respawn:/sbin/getty -L ttyS0 115200 linux' >> /etc/inittab
    sed -i 's/^\([1-6]:.* tty[1-6]\)/#\1/' /etc/inittab

Disable RAMTMP

    echo RAMTMP=no >> /etc/default/rcS
    echo RAMSHM=no >> /etc/default/rcS
    echo RAMLOCK=no >> /etc/default/rcS

Finally, set root password

    passwd

Now we're done here

    exit
    umount /media/debian/proc
    umount /media/debian/dev/pts

That's it. You can now copy the files to your USB flash and run Debian on Stora.
