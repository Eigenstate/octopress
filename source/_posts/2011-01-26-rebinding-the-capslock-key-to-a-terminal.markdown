---
author: Robin Betz 
comments: true
date: 2011-01-26 05:54:57+00:00
layout: post
slug: rebinding-the-capslock-key-to-a-terminal
title: Rebinding the CapsLock Key to a Terminal
wordpress_id: 13
categories: Tutorial
---

If you're anything like me, the Caps Lock key annoys the heck out of you. You're typing along, not looking at your keyboard, when all of a sudden everythinG LOOKS LIKE THIS. Nevertheless, the key is in a convenient location (as evidenced by its often accidental pressing). I've found it immensely convenient to bind the key instead to **Yakuake**, a wonderful Quake-style terminal emulator for KDE. Now, whenever I press the Caps Lock key, a terminal pops down from the top of the screen, and I can press once more to banish it. It's extremely useful if you're on a Linux system and want easy access to the command line.

What follows is how to set this up on your own machine.

<!-- more -->


	
  * You must be running Linux of some flavour. Distribution doesn't matter, as all you need is a working installation of X. If you have a GUI of some sort, you're fine.

	
  * Download and install the slide-down terminal that is relevant for your setup. I use [Yakuake](http://yakuake.kde.org/) in KDE, and recommend it highly. For GNOME, [Tilda](http://sourceforge.net/projects/tilda/) seems to be roughly equivalent, although I can't vouch for its quality.

	
  * Put the following code somewhere that it will be run on startup. I recommend editing ~/.bashrc. This will remove the functionality of the Caps Lock key.




<blockquote>

>     
>     xmodmap -e "remove lock = Caps_Lock"
> 
> 
</blockquote>





	
  * Now, give the key its new purpose. Start up Yakuake (or Tilda, for GNOME people), and go to settings. Change the default close/retract key from F12 (or F2 for Tilda) to Caps Lock.

	
  * [![Yakuake Hotkey Selection](http://compilingsheep.files.wordpress.com/2011/01/yakuake.jpg?w=300)](http://compilingsheep.files.wordpress.com/2011/01/yakuake.jpg)

	
  * If desired, set the terminal emulator to start at login. For KDE, go to System Settings->Advanced->Autostart and hit the "Add Program" button. Enter yakuake when prompted. GNOME should follow a similar procedure.


Hopefully this will be as effective and useful for you as it is for me. (It's also pretty useful for impressing people)
