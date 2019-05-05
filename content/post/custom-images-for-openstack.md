---
title: Creating Custom Images for Openstack
date: 2014-01-19 18:00
draft: false
tags:
  - Tutorials
image: images/drives.jpg
aliases:
  - /blog/2014/01/19/creating-custom-images-for-openstack/
---

## Installing the operating system with KVM

Machine images for the cloud infrastructure are nothing more than a virtual machines' hard drive, not much different than what you would use with VirtualBox, Parallels, or VMWare.  This example will demonstrate how to create an Ubuntu 11.10 server image using KVM.This was completed using a Redhat based desktop system.

<!--more-->

First off, create a 4GB virtual hard drive that we will install to.

```sh
dd if=/dev/zero of=new.img bs=1M count=4096
```

Download the 'iso' installation image that you would normally bun to a disk for the installation and 'boot' the installation media.

```sh
kvm -cdrom ubuntu-11.10-server-amd64.iso -drive if=scsi,file=new.img,boot=off
```

or

```sh
qemu-kvm -cdrom ubuntu-11.10-server-amd64.iso -drive if=scsi,file=new.img,boot=off
```

The virtual system will boot into the installer and you will be presented with the installer screen.  Go through the options and select the appropriate language and keyboard options.  Let the network and hardware setup be automatically configured.  Once you get to the partitioning and formatting stage, select the 'Guided (use entire disk)' option.  Make sure you do not use LVM!!!  Select your disk and let the installation begin.

Once the base installation has completed, you will be asked to set up a new user.  For the sake of this tutorial, let's name the user "Cloud User", with the username "cloud" and the password "password".  There is no need to encrypt the new uses home directory.  You will then be asked about system updates.  Select 'No automatic updates'.  Add the SSH server option from the next screen.

The installer will then install the system packages and will exit.  This could take a great deal of time depending on your workstation and whether or not you have VM hardware acceleration enabled in the BIOS.  Once the install is complete, exit and close out the window.  At this point you will have an image that can be used with KVM, to make it compatible with the cloud, we need to make a few more adjustments.

## Extracting the root file system

Unlike a standard install for a VM or server/desktop, we only need the files that were installed to the 'Root' filesystem.  We don't need the boot or swap partitions that the installer automatically creates, and the next steps will extract the partition that we need for the upload.  Use the 'parted' command to find the size of the partition and where it starts (in blocks):

![parted on a terminal](/images/parted-1.jpg)

In parted, the 'u'nits are selected as 'b'locks and the partition information is 'p'rinted out.  We can tell by the partition information that the root file system starts at block 1048576 and is 4048551936 blocks in length.  Use those numbers to extract the partition using 'dd'.

```
dd if=new.img of=ubuntu-server-11.10.img bs=1 skip=1048576 count=4048551936
```

This will take a long time as it is copying one block at a time.  To significantly speed things up you can copy in 512 block segments.  Divide both the starting point and the size by 512 and run:

```
dd if=new.img of=ubuntu-server-11.10.img bs=512 skip=2048 count=7907328
```

Your root filesystem has now been extracted.

## Extract the kernel and ramdisk

In order to boot your machine image, we need two more files, namely the kernel and the ramdisk that will be used by the bootloader to load drivers, set up devices and transition to the machine image that you are creating.  If you have done this before, this is not a required step, as they are (somewhat) interchangeable and you can use previously a extracted kernel and ramdisk.  Start out by setting up a loopback device.  Mount the image and copy out the files.

```
losetup /dev/loop5 ubuntu-server-11.10.img
mount /dev/loop5 /tmp/image
cp /tmp/image/boot/initrd* ./ubuntu-server-11.10-ramdisk
cp /tmp/image/boot/vmlin* ./ubuntu-server-11.10-kernel
umount /tmp/image
losetup -d /dev/loop5
```

## Customize the image

At the very least you will need to install the 'cloud-utils' package for ubuntu installs and adjust a few of the startup scripts for redhat based images in order for the networking and user data to be properly configured and read.  start by creating a loopback device and mounting the image that you just created.  If you find that you will be doing this often it might be a good thing to put this into a script.

```
mkdir /tmp/image
losetup /dev/loop5 ubuntu-server-11.10.img
mount /dev/loop5 /tmp/image
```

Now bind some of the special purpose system directories to the device.

```
mount -o bind /proc /tmp/image/proc
mount -o bind /sys /tmp/image/sys
mount -o bind /dev /tmp/image/dev
```

Copy the resolv.conf file into the image so we can access the network while we visit.

```
cp /etc/resolv.conf /tmp/image/etc/resolv.conf
```

We are ready to chroot into the new environment. If you have ever used Gentoo Linux before this entire process should be looking very familiar.

```
chroot /tmp/image /bin/bash
```

You are now in the 'image' just as you would have been if it was live and you had logged in.  From here, make changes, install applications and modify files.  At the very least you should perform an apt-get install of cloud-init if you are working with ubuntu or debian.  For any flavor of linux you will need to adjust /etc/fstab to mount the root filesystem by label in case openstack resizes the image based on instance types.  This can make the UUID used to mount the root filesystem in newer distributions invalid.  Change:

```
UUID=e7f5af8d-5d96-45cc-a0fc-d0d1bde8f31c / ext4 errors=remount-ro 0 1
```

to

```
LABEL=vm-rootfs / ext4 defaults 0 0
```

Once you are done making changes and installing applications, you can exit and clean up

```
rm /tmp/image/etc/resolv.conf
rm /tmp/image/etc/udev/rules.d/70-persistent-net.rules
umount /tmp/image/dev
umount /tmp/image/proc
umount /tmp/image/sys
umount /tmp/image
losetup -d /dev/loop5
```

Lastly, change the filesystem label on the image to work with the stab entry we previously set up.  Once this is complete, you are done.  Upload your the machine image, kernel and ramdisk and spin up your instances.

```
tune2fs -L vm-rootfs ubuntu-server-11.10.img
```
