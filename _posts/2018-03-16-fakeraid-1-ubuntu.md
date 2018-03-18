---
layout: post
title: Installing Ubuntu Server 16.04 on Hardware Backed Software RAID1 (FakeRAID)
excerpt_separator:  <!--more-->
categories:
    - technology
tags:
    - ubuntu
    - raid
    - software raid
    - troubleshooting
    - server
    - hardware
    - software
---
I encountered an issue booting Ubuntu Server 16.04 when installing it on a SuperMicro server that had been configured with Software RAID 1. The operating system install script would detect that a RAID environment was active and it would install correctly, but when booting into the newly installed system I would be presented with a blank screen and a blinking cursor. Grub wasn't even loading.

To solve this issue I booted into a live CD of Ubuntu 16.04 and did the following from the terminal:

```bash
sudo su
mount /dev/mapper/<name of your raid partition> /mnt
cd /mnt
mount -t proc proc proc/
mount -t sysfs sys sys/
mount -o bind /dev dev/
mount --rbind /run run/
chroot .
grub-install /dev/sda
grub-install /dev/sdb
update-grub
exit
reboot
```
**Note: Your /dev/sda and /dev/sdb might be different, please use the disks that were used to create the RAID drive**

After restarting I was now greeted by the familiar Grub splash screen and Ubuntu proceeded to boot correctly.

**_P.S. I think it's just better to use software RAID through Linux anyway, since FakeRAID is kind of the worst of both hardware and software implementation. ;)_**