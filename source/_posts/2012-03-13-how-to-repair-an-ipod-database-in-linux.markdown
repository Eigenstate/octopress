---
author: Robin Betz 
comments: true
date: 2012-03-13 05:10:33+00:00
layout: post
slug: how-to-repair-an-ipod-database-in-linux
title: How to repair an iPod database in Linux
wordpress_id: 33
categories: Tutorial
---

iPods, especially iOS devices, have often been a pain to manage without iTunes. For the longest time I had all my music on my Windows partition, and the only reason I would ever boot Windows is because I got a new album that I wanted to sync. Thanks to a wonderful combination of software, I never need to leave my nice command line / superior in every way Linux interface.

<!-- more -->

## How to mess up your iPod pretty badly


A couple days ago I booted Windows to game (alas, the final frontier for Linux) and my iPod Touch happened to be plugged in. iTunes started syncing it, but I had modified the database in Linux to add my own music. The result? A confused iPod that listed songs as being there but would refuse to play any of them.

I could restore the entire thing in iTunes, but would have to jailbreak again, find all my custom settings and scripts, and copy over loads of music. With trial and error, I managed to find a combination of software that let me fix the problem in Linux. Please read through the whole thing and try to actually understand what each step does before running around as root and deleting things!

You **do not **need to have a jailbroken device for any of these steps. These steps work for me on a first generation jailbroken iPod Touch, your results may vary.


## How to fix that


I use [libimobiledevice](http://www.libimobiledevice.org/) integration in [Clementine](http://www.clementine-player.org/), my music player of choice, to normally manage my iPod touch, which gives me a drag and drop interface and automatic syncing. To repair, though, you'll have to actually mount the device.

libimobiledevice includes a package called ifuse (Gentoo users, Portage lists this as a separate ebuild) that will detect and mount iPods. Attach the device and specify a mountpoint (I'll use /mnt/ipod) by running the following **as root**. The allow_other option allows you to access the device for the other steps as a normal user.

    
    ifuse -o allow_other /mnt/ipod


Now, to actually repair the database which has been all kinds of ruined, we will delete it and then regenerate it. There probably is a more elegant way to do this that does not involve deleting all your music, but unfortunately this method will do exactly that. You won't lose anything else though, and your jailbreak / settings / apps will be fine. Delete everything in the iTunes database folder:

    
    rm -r /mnt/ipod/iTunes_Control/iTunes/*


If you have no idea what you're doing and delete things in the iTunes_Control/Device folder (like I did when experimenting initially), go get yourself a new HashInfo file [here](http://ihash.marcansoft.com/) and put it in that directory. We'll regenerate the SysInfo and SysInfoExtended files there later.

Now, delete all your music :( I don't know if this is actually necessary, but if you skip this step make sure that all files on the device actually appear in its GUI and don't remain orphaned. When you sync, mp3 files are renamed and go into the F00, F01, etc folders in iTunes_Control/Music. Delete all those.

    
    rm -r /mnt/ipod/iTunes_Control/Music/*




Now that we have deleted loads of things, it's time to actually regenerate an empty database and then populate it. If you regenerated the HashInfo file, generate the SysInfo and SysInfo extended files by first finding the device UUID:




    
    lsusb -v | grep iSerial




And then using the long, 40-digit string from that output to replace <uuid> below




    
    ipod-read-sysinfo-extended <uuid> /mnt/ipod




For this, we'll use [gtkpod](http://www.gtkpod.org/wiki/Home), which is a wonderful utility for all sorts of iPod management stuff. Start it up, and under Edit->Configure Repositories point to your iPod's mountpoint, in this example /mnt/ipod, and indicate the model. It will tell you that the directory structure is invalid, which is correct since we just deleted all of the folders in Music, so click the "Correct Directory Structure" button. Gtkpod will proceed to regenerate the database files.




Copy a few songs with gtkpod just to make sure the database is good. Now, use your method of choice to restore the rest of your music.




Congratulations! You hopefully now have a fully functional iPod with a minimum of effort!



