# kng_re-openwrt
OpenWrt for ZyXEL Keenetic Giga III

Wifi works. For some reason my flash has too many bad sectors inside
"rootfs_data" partition, so I have moved rootfs to ubifs, which makes the
installation procedure somewhat quirky.

First of all you need to build kernel+initramfs and squashfs images. When you are
done, hook up a serial console, choose "1: Load system code to SDRAM via TFTP.",
start a TFTP server on your PC (dnsmasq will work just fine), serve an initramfs
image to your router and wait until you get a shell.

While you are waiting for the TFTP transfer to complete, separate rootfs and
initramfs parts of the combined image:

    F=bin/ramips/openwrt-ramips-mt7621-mt7621-squashfs-sysupgrade.bin
    O=$(grep -obUaP 'hsqs$' $F | cut -d ':' -f 1)
    dd if=$F of=kernel.bin bs=$O count=1
    dd if=$F of=root.squashfs bs=$O skip=1

Then, prepare an UBI filesystem for rootfs on your router:

    ubiformat /dev/mtd9
    ubiattach -m 9
    ubimkvol /dev/ubi0 -N rootfs -m
    mkdir -p /mnt/rootfs
    mount -t ubifs /dev/ubi0_0 /mnt/rootfs

And populate it:
  
    cd /tmp
    mkdir -p /mnt/squashfs
    wget http://pc/openwrt/rootfs.squashfs
    opkg update && opkg install squashfs-tools-unsquashfs
    unsquashfs -d /mnt/rootfs root.squashfs
    umount /mnt/rootfs
    ubidetach -d 0
    
The last thing you need to complete the process is writing the
kernel on the MTD:

    wget http://pc/openwrt/kernel.bin
    mtd erase kernel
    mtd write kernel.bin kernel
    reboot
