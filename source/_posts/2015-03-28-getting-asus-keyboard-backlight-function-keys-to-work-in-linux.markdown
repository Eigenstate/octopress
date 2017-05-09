---
layout: post
title: "Asus UX303 in Linux: Part 1 - Keyboard backlight function keys"
subtitle: "Keyboard backlight function keys"
date: 2015-03-28 21:06:05 -0700
comments: true
categories: [Tutorial, UX303, Asus]
published: true
---

My new Asus ux303 laptop is so new that getting all functionality under Linux can take a bit of effort. This is the first of a series of posts designed to help you out if you are in the same boat, either with this same model of laptop or similar.

<!-- more -->

## The problem ##
Linux is getting a lot better on modern hardware. Most of the function keys on the keyboard work just fine out of the box, including screen brightness, airplane mode, volume, and hibernation. However, the Fn+F3/F4 keys that control the brightness of the keyboard backlight don't do anything. Let's fix that.

## A bit about /sys ##
If you have an LED (or really any piece of hardware) that the kernel knows about, it's going to have an entry in [sysfs](https://www.kernel.org/doc/Documentation/filesystems/sysfs.txthttps://www.kernel.org/doc/Documentation/filesystems/sysfs.txt), which is the filesystem mounted at /sys that links kernel data structures to userspace, with every kobject getting a directory. [More here](http://www.win.tue.nl/~aeb/linux/lk/lk-13.html)

More simply, all devices on your system that are built in to your kernel (or loaded as modules) will have a folder in /sys that gives you some information about them. Let's look at the sysfs entry for the backlight led, for example, which will be present in /sys/class/leds.

I have two symlinks in this folder, named asus::kbd\_backlight and phy0-led. Yours will probably vary based on your hardware. Looking in the asus::kbd\_backlight folder, I see the following:

      $ ls /sys/class/leds/asus::kbd_backlight


      total 0
      -rw-r--r--. 1 root 4096 Mar 29 18:22 brightness
      lrwxrwxrwx. 1 root    0 Mar 29 20:45 device -> ../../../asus-nb-wmi
      -r--r--r--. 1 root 4096 Mar 29 20:45 max_brightness
      drwxr-xr-x. 2 root    0 Mar 29 20:45 power
      lrwxrwxrwx. 1 root    0 Mar 29 20:45 subsystem -> ../../../../../class/leds
      -rw-r--r--. 1 root 4096 Mar 29 20:45 trigger
      -rw-r--r--. 1 root 4096 Mar 29 20:45 uevent

There's a couple of useful files here. First, the **max\_brightness** file contains the maximum allowable brightness value, and the **brightness** file contains the current brightness value.

Currently the backlight is off:

      $ cat max_brightness brightness
      3
      0

To change that, (as root),

      sudo echo 2 > brightness

And the keys should light up.


## Writing a script to change brightness ##
Let's create a basic script to alter the brightness of the LEDs by changing the brightness entry in /sys.

{% codeblock lang:bash /usr/bin/keyboard_backlight %}
#!/bin/bash

if [[ -z $1 ]]; then
  echo "Usage: keyboard_backlight up/down"
  exit 1
fi

max=$(cat "/sys/class/leds/asus::kbd_backlight/max_brightness")
brightness=$(cat "/sys/class/leds/asus::kbd_backlight/brightness")

if [[ "$1" == "up" ]]; then
  if [[ "$brightness" -lt "$max" ]]; then
    echo "$((brightness+1))" >> /sys/class/leds/asus\:\:kbd_backlight/brightness
    fi
elif [[ "$1" == "down" ]]; then
  if [[ "$brightness" -gt 0 ]]; then
    echo "$((brightness-1))" >> /sys/class/leds/asus\:\:kbd_backlight/brightness
    fi
fi
{% endcodeblock %}

Note that you'll need root permissions if you want to save it in /usr/bin, but you can put it wherever you want, it does not need to be in $PATH.

## Getting permissions correct ##
But wait, don't I have to be root to change the stuff in /sys? Yes, by default the permissions are such that an ordinary user can't mess with any of the settings here, which makes sense given that you don't want just anybody to be able to turn off the screen on your mainframe, etc.

Try running your script without sudo and you will get a nice permission denied error.

A basic "hack" around this issue would be to put an exception in /etc/sudoers to allow the script to run with sudo without a password, but this is dangerous for a variety of reasons. 

A much better solution is to allow unprivileged users to change the brightness of the LEDs. This is where udev comes in.

[Udev](https://www.kernel.org/pub/linux/utils/kernel/hotplug/udev/udev.html) manages the device nodes in /dev and the information in sysfs, and handles the addition and removal of devices. It operates by a series of rules that tell it what entries to create in /dev, what to name them, and what the permissions should be.

 We will create a udev rule that will set the permissions on the LED brightness file so that it is writeable by regular users.

I love love love udev rules, but don't want to get into a ton of detail on how to write them, as there are much better guides available, such as [this one](http://www.reactivated.net/writing_udev_rules.html) by Daniel Drake. 

Make the following file.

{% codeblock lang:bash /etc/udev/rules.d/99-keyboard-leds.rules %}
DEVPATH=="/devices/platform/asus-nb-wmi/leds/asus::kbd_backlight", RUN+="/bin/chmod 0666 /sys/class/leds/asus::kbd_backlight/brightness"
{% endcodeblock %}

If you have a different setup, get the value for DEVPATH from the output of
    
    udevadm info -q all /sys/class/leds/asus\:\:kbd_backlight

Or whatever the sysfs entry for your backlight is.

_Note_: This is kind of a hack. I really should be using the MODE attribute but it is unclear how to get that to apply to sysfs entries rather than just /dev symlinks. If I find a better way I will update.

What this rule does is run chmod on the brightness file when the backlight LED sysfs entry is created so that everyone can write to it. For my personal laptop, this isn't security concern as I am the only user and if someone manages to change my LED brightness I don't really care.

You will have to reboot or unplug and re-plug a device for udev rules changes to take effect, so in this case reboot to see the permissions get fixed.

## Mapping to function keys ##
Now that we have a script that any user can run to change the backlight, it's time to make pressing the appropriate function key invoke the script with the correct argument.

For this we will use [xbindkeys](http://www.nongnu.org/xbindkeys/xbindkeys.html), which can map keyboard and mouse actions to shell commands independent of window manager.

Install through your package manager, then run

    xbindkeys -d > ~/.xbindkeysrc

To create a blank configuration file in your home directory.

To find the keycodes for the backlight function keys, run

    xbindkeys -k

and type the key combinations (for me, Fn+f3 and Fn+f4) into the little window. If you chose to put your configuration file somewhere other than your home directory, you will need to specify its location with the -f option.

The output will consist of a raw keycode and perhaps a command. In my case, I got XF86KbdBrightnessUp and XF86KbdBrightnessDown as X has a pretty decent idea of my keyboard layout.

Now edit your config file so the keyboard\_backlight script is associated with those keycodes:

{% codeblock lang:bash ~/.xbindkeysrc %}
"/usr/bin/keyboard_backlight up"
  XF86KbdBrightnessUp

"/usr/bin/keyboard_backlight down"
  XF86KbdBrightnessDown
{% endcodeblock %}

Then set xbindkeys to start when your window manager does, through whichever method you prefer (I put it in my i3 config file).

## Great you're done ##
Congratulations, now you can type in the dark.

