---
author: Robin Betz 
comments: true
date: 2011-05-06 19:47:04+00:00
layout: post
slug: easy-safe-backups-with-rsync-and-zfs
title: Easy, Safe Backups with rsync and ZFS
wordpress_id: 23
categories: Tutorial
---

If you're anything like me, you worry about losing your data and want to backup, but you're too lazy to wait for everything to copy to a portable hard drive/ten thousand CDs/Dropbox/whatever. As a result, you rarely, if ever, run backups, and when your hard drive finally dies or you break your system somehow, you're left with very outdated backups that are impossible to put back on your computer or nothing at all.

This is especially true if you run a bleeding-edge system like Gentoo, do kernel hacking, or enjoy running alpha builds to get new features- you will inevitably run into something that will break your system, and then in trying to fix it things only get worse. Backups then become important in not only preserving your data, but also serve to save your system in a stable state so that it may be restored easily if something breaks. Simply just copying your files onto a flash drive isn't going to cut it.

I've managed to find a system that works quite well for me, and has actually saved me quite a bit of time in the past. It uses an external hard drive, rsync, and the ZFS file system to do incremental backups that preserve everything from ownership to permissions to dynamic linking. First I'll tell you why this system is awesome, and then how to set it up yourself.

<!-- more -->


## **What's ZFS?**


ZFS is a combined file system and logical volume manager. File systems are ways to structure information on your hard drive- Windows uses NTFS, Macs use HFS+, Linux uses either ext3 or ext4 file systems in a standard setup. ZFS doesn't actually stand for anything nowadays.

ZFS was made for backups. It was designed to prevent against bit rot and silent data corruption, and, at the cost of some storage space, does an excellent job detecting errors in reading or writing the data, making it invaluable for backing up. ZFS' abilities to detect file system corruption will catch errors that are invisible on other file systems (at least until they build up and everything breaks). Using ZFS has enabled many users to detect faulty hard drives or even [damaged power supplies](http://blogs.sun.com/elowe/entry/zfs_saves_the_day_ta).

Since I run GNU/Linux, this guide will focus on using ZFS on Unix-like systems. However, please note that ZFS was developed by Sun Microsystems, and is licensed under the Common Development and Distribution License (CDDL) which is incompatible with the GNU GPL, preventing incorporation of this filesystem directly in the kernel. However, the File System in Userspace (FUSE) kernel module includes ZFS support. **Update:** There actually is a kernel module that gets around the licensing problem without using FUSE [here](http://zfsonlinux.org/faq.html). I haven't tried building it though, and suspect it might be hard to compile. Also kernels newer than 2.6.36 are not supported, so if you want your 2.6.38 scheduling, you're out of luck.


## **And rsync?**


Rsync is a program that synchronizes files and directories with minimal data transfer. It is capable of preserving permissions, ownership, and even access times. Rsync makes incremental backups possible- after running an initial backup that copies everything to disk, only the changes from the original need to be synced after that, resulting in much faster backups that can be accomplished silently in the background. Rsync is free software and is in all Linux distributions.


## **The Setup**


This will be loosely structured as instructions, however I would recommend you read the whole thing before embarking.


### 1. Installing necessary software


I don't know if standard (ones you didn't compile yourself) kernels include the zfs-fuse module. Follow the instructions to download and install from the ZFS-FUSE [website](http://zfs-fuse.net/) if you don't have it, and then load the module with modprobe. Gentoo users, emerge zfs-fuse and recompile your kernel (it should be under filesystems) to get it. It's masked, but I have never had any problems. I'll assume you know how to do this, if not there are plenty of guides elsewhere or you can ask for help in the comments.


### 2. Prepping your hard drive


Obtain an empty external hard drive that is at least _twice _the size of the device(s) you wish to back up. This is extremely important since ZFS can only detect filesystem errors in comparison to something else. This is why ZFS partitions are mirrored-- two identical copies of your data will be stored and checked against each other for mistakes. Yes, it takes double the space, but a 2TB external hard drive can be obtained for less than $100, and your data are worth it. If you decide not to mirror, ZFS can detect but not correct any filesystem errors it finds, which is absolutely useless from a data integrity standpoint.

Partition your drive into two equally-sized partitions using fdisk (if you like the command line) or gparted/whatever GUI tool you like. Again, there are many guides on this elsewhere. Make sure you know where your disk is in /dev. You can set it to automatically mount at the same point every time using [udev](http://www.reactivated.net/writing_udev_rules.html), but this is not necessary (a guide on udev rules is planned). However, it is _very important_ that you do not confuse your disk at, say, /dev/sdb1 with your home video hard drive at /dev/sdc1, otherwise ZFS will nuke all the data on the wrong disk. If when you partitioned the disk you gave it a label, looking in /dev/disk/by-label/ can often be helpful.

Create a[ zpool](http://www.manpagez.com/man/8/zpool/). This is the zfs unit that holds all of your backup data. For this guide, the two partitions will be at /dev/backup1 and /dev/backup2. Substitute your own partition mount points here! Otherwise it will not work. We'll make backup1 and backup2 mirror each other, and we'll name the pool awesome_backup.

    
    zpool create awesome_backup mirror /dev/backup1 /dev/backup2


Now that the two partitions are set up, create the actual ZFS filesystem to hold your backup. Mount the device somewhere (it'll be at /mnt/backup/ for this example). We'll name it computer, and it'll look like a directory. Set compression to be on.

    
    zfs create /mnt/backup/computer
    zfs set compression=on /mnt/backup/computer


If you want to have multiple directories (for example, separate ones for your /home directory and another one for all your binaries), you can run this command again. To show the existing filesystems and their properties:

    
    zfs list


and you'll see something like this. This is my setup- I have separate filesystems for backing up my Windows system and my Linux one. The Windows system is where I keep all my music, and I back it up through Linux using this exact method, I've just mounted the NTFS partition. Don't worry about it for now.

    
    NAME               USED  AVAIL  REFER  MOUNTPOINT
    backup             141G   339G  17.8G  /mnt/backup
    backup/Whitfield  35.1G   339G  23.1G  /mnt/backup/Whitfield
    backup/Windows    88.4G   339G  88.4G  /mnt/backup/Windows


Now you have a working zfs filesystem! By default, keep it unmounted so that it can't be accidentally changed.

    
    zpool export awesome_backup




### 3. Doing the backup


Now that the ZFS filesystem is up and ready to go, we can actually do backups on it. First, import the filesystem so you can access it.

    
    zpool import awesome_backup


To backup, say, your home folder into your computer filesystem, run:

    
    rsync --delete -av /home/ /mnt/backup/computer/home/


Rsync is the program used to do the backup. The --delete deletes extraneous file from the destination directory. Don't worry, your home directory is unchanged. The -a option archives, which means it preserves permissions, dates, and several other attributes that make the resulting copy an identical clone of the system. The -v option means verbose, which will display every item that is being synced. The first folder listed is the source folder, and the second is the destination. I'd suggest read the manpage for rsync if you want more information.

You can run this for any folder you want to backup, or all of them. The first time you run it it will take quite some time as every file needs to be copied, but subsequent runs will be much faster as only files that have been changed will be altered.


### 4. Recovering data


Let's say you decided to suddenly recompile your system with the testing branch of every package because you are dumb, and as expected it became massively broken (true story, this happened). Fortunately, you don't have to re-install and start from scratch. Just use your backup! Reverse the source and destination folders and don't delete extraneous files (they're new ones on your computer that haven't been backed up yet).

    
    zpool import awesome_backup
    rsync -av /mnt/backup/computer/home/ /home/
    zpool export awesome_backup


You can do this to restore everything from your entire system to the individual config files.


### 5. Snapshots and Clones and gosh oh my


There are loads more options for using your zfs system that I may describe later. Nevertheless, before you start on making your own ZFS backup system, I highly recommend you read the man pages for zfs, zpool, and rsync at minimum. This [page](http://www.gentoo-wiki.info/Backup/Backup_using_ZFS) was responsible for the initial idea for this guide, although some of it is out of date. As always, comments and questions are very welcome.
