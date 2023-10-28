# Boot a Raspberry Pi 4 from an LVM Root Partition

I wanted to boot to an LVM root partition on my USB drive and looked around the Internet.  The examples given were very helpful to me understanding what to do, but they were out-of-date with the lastest Raspberry Pi OS, Bookworm.

This documents how I configured booting from a USB drive with the root on an LVM volume using Raspberry Pi 4, and Raspberry Pi OS Lite (64-bit) (Bookworm).

You could do the same thing booting from a MicroSD card.  Just tweak the instructions.  If re-using the same SD card, you'd want to redo the partions and reformat them (including LVM steps) after `tar`ing the contents.

## Assumption

This readme assumes you are familiar with Raspberry Pi installation and that you have all the peripheral equipment to do the installation such as power supply, case, keyboard, and display.

# Equipment

All of the work here was done on an Ubuntu 22.04 desktop and a Raspberry Pi 4B.  On the desktop, I had an available USB port for the USB drive and a MicroSD card reader (both of these were provided by an external USB-C dock).

The MicroSD card was a 32GB Samsung Evo Plus.  The USB drive was a 1 TB Seagate USB HDD.

You'll need a keyboard and monitor for the initial Raspberry Pi OS installation unless you configured all the extra settings including SSH and networking in the `rpi-imager` creation of the image (by clicking the gear icon).

# Tools

The follownig tools were used.

## rpi-imager

Installed on the desktop via `apt-get install rpi-imager`.

## LVM2

Installed on the desktop, and eventually the pi, via `apt-get install lvm2`.

# Steps

## Device Prep

1. Plug both MicroSD and USB drive into Ubuntu desktop.  For this example, `/dev/sda` is the MicroSD and `/dev/sdb` is the USB drive.
2. Using `parted`, remove all partitions on both devices.  **THIS DESTROYS ALL EXISTING DATA**
3. On the USB drive, create a 500MB FAT32 partition and the remainder of the disk as an LVM partition:
```
parted /dev/sdb mkpart primary fat32 2048s 512MiB
parted /dev/sdb mkpart primary ext4 512MiB 100%
parted /dev/sdb set 2 lvm on
```
4. Format and label the FAT partition: `mkfs.fat -F 32 -n bootfs-rpi /dev/sdb1`
5. Setup LVM on the USB drive, create and format the root volume:
```
pvcreate /dev/sdb2
vgcreate usb-rpi /dev/sdb2
lvcreate -L 30G -n rootfs usb-rpi
mke2fs -t ext4 -L rootfs-rpi /dev/usb-rpi/rootfs
```
## Initial install of Raspberry Pi
1. Launch `rpi-imager`
2. Choose OS, Raspberry Pi OS (other), Raspberry Pi OS Lite (64-bit)
3. Choose Storage, select the MicroSD device
4. Optionally set other options by clicking the gear icon.
5. Write
6. After safely ejecting the MicroSD, place it in the Raspberry Pi, boot, and follow the installation.
7. Once the OS installation is complete, install `LVM`.  Look at the output of the `lvm2` install and note that there is a hook that updates the `initramfs` image -- so `LVM` is automatically included in the Init RAM Disk!!!  This is a significant improvement from previous examples.  It must have been an update in Raspberry Pi OS.
```
sudo apt update
sudo apt-get install lvm2 -y
```
8. **DO NOT MAKE ANY OTHER UPDATES AT THIS TIME**
9. Safely shutdown the pi with `sudo shutdown now`

## Copy boot and root partitions to USB drive
1. Remove the MicroSD and place it back in the desktop.
2. Mount the SD card partitions (assuming it is `/dev/sda`) and archive the contents:
```
sudo -Es
mkdir -p rpi/boot/firmware
mount /dev/sda2 rpi
mount /dev/sda1 rpi/boot/firmware
tar -cvzf rpi.tar.gz -C rpi ./
umount /rpi/boot/firmware
umount /rpi
```
3. Mount the USB partitions and restore the contents.  **Leave them mounted for now**:
```
mount /dev/usb-rpi/rootfs rpi
mount /dev/sdb1 rpi/boot/firmware
tar -xvzf rpi.tar.gz -C rpi
```

## Point to the new partitions
1. In `rpi/boot/firmware/cmdline.txt`, update the `root=` entry to be `root=LABEL=rootfs-rpi`.  This will cause the root partition to be loaded from our LVM rootfs volume.  Since `LVM` is a part of the Init RAM Disk image now, the LVM volume will be usable during the boot sequence.
2. In `rpi/etc/fstab`, update the entries for `/` and `/boot/firmware` to `LABEL=rootfs-rpi` and `LABEL=bootfs-rpi` respectively.

## Safely eject the USB drive from the desktop.
1. Unmount the partitions:
```
umount rpi/boot/firmware
umount rpi
rmdir rpi
```
2. Safely eject (NOTE:  You shouldn't need to deactivate the LVM volumes to safely eject).

## Plug it in and boot!
1. Plug the USB drive into the Raspberry Pi
2. The MicroSD should not be in the Pi
3. Boot up!
4. Optionally use `raspi-config` to set the boot order to be USB drive first.

# Reference

[Answer to "Easy backups and snapshots of a running system with LVM" on Stack Exchange](https://raspberrypi.stackexchange.com/a/85959)

