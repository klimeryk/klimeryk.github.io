---
title: 'Avoiding Kernel Modules When External Firmware Is Needed'
date: Mon, 05 May 2014 21:41:00 +0000
draft: false
url: /2014/05/05/avoiding-kernel-modules-when-external-firmware-is-needed/
tags: ['Kernel', 'Linux']
---

If you’re just interested in keeping your kernel module-free and be able to watch funny cat videos on youtube, skip to the [solution](https://klimer.eu/2014/05/05/avoiding-kernel-modules-when-external-firmware-is-needed/#modules-solution).

I’m an avid user of the [Gentoo](http:/www.gentoo.org/) flavour of Linux, specifically the [Hardened profile](http://wiki.gentoo.org/wiki/Hardened_Gentoo). As you can imagine, I like to think of myself as a security-aware user. With that in mind, I prefer to keep my kernel module-free (by actually disabling the ability to load kernel modules), building in every driver that I need and keeping all the rest out. The reason for this, from a security point of view, is that loading a malicious kernel module is very often \[citation needed\] the second step after a successful privilege escalation. I don’t need module loading – I know my hardware, it doesn’t change that often.

So imagine my disdain, when I was setting up my laptop, a (t)rusty Lenovo ThinkPad X220, and it turned out that the WIFI card (an Intel Centrino something, supported by the iwlwifi driver) needed to load external firmware (residing in /lib/firmware) upon load. Unfortunately, this did not work when the driver was built in into the kernel – I presume that /lib/firmware is simply not available when the driver loads. A possible solution would be to create an initrd image and put the firmware there… but that seemed too much hassle. So I built the driver as module and wept in the corner that night…

Fast forward to today, when I was setting up another ThinkPad (E540, for my parents) and I’ve stumbled upon the same situation – an Intel WIFI card (Intel Wireless-N 7260) that needed external firmware. But this time I’ve decided to be a man-, nay – a professional!, and tackle this problem. I mean, this the Linux Kernel, we don’t need to take this shit!

The solution was far easier than I thought, requiring no hacks or custom scripts. Didn’t need to Google even, just looked at the available options. Here are the steps:

*   Let’s assume you have a working WIFI card, the firmware is in `/lib/firmware` and is named `iwlwifi-7260-8.ucode`. If not, install the appropriate package from your distro’s package manager or download the firmware directly (for example, for Intel cards, from [http://wireless.kernel.org/en/users/Drivers/iwlwifi](http://wireless.kernel.org/en/users/Drivers/iwlwifi)).
*   Head on to your kernel’s sources (you do build your own kernel, right?), usually `/usr/src/linux`.
*   `make menuconfig`
*   Let’s start off with building in the driver we’ll be including the firmware for. For me, it was **Intel Wireless WiFi Next Gen AGN - Wireless-N/Advanced-N/Ultimate-N (iwlwifi)** (`CONFIG_IWLWIFI`).
*   Now, the options we are interested in reside in **Device Drivers -> Generic Driver Options**.
*   You probably have **Userspace firmware loading support** (`CONFIG_FW_LOADER`) enabled already (your WIFI driver should autoselect it), if not – enable it.
*   Under it, select the **External firmware blobs to build into the kernel binary** (`CONFIG_EXTRA_FIRMWARE`) option. As the help says: This option allows firmware to be built into the kernel for the case where the user either cannot or doesn’t want to provide it from userspace at runtime. (…) This option is a string and takes the (space-separated) names of the firmware files.
*   So, let’s do as the f…reaking manual tells us to do, set the value to your firmware’s filename (separating the files with spaces, if more than one). For our example, I’ve set it to `iwlwifi-7260-8.ucode`.
*   Setting this value enabled another option beneath it: **Firmware blobs root directory** (`CONFIG_EXTRA_FIRMWARE_DIR`). This is the dir that will be searched for the firmware files we’ve specified above. It defaults to the firmware directory in the kernel’s source tree. The easiest way is to point it to `/lib/firmware` (as I’ve done), just remember that a future kernel upgrade might mean that you’ll need to update the firmware files as well. Keeping the firmware files in the kernel’s source tree is safer, but requires you to remember (OH NOES!) to copy over (or download updated versions) every time you compile a new version of the kernel.
*   Save configuration, execute `make -j4` (where 4 is the number of CPU cores you have, to speed things up a bit), etc.
*   Reboot and check in dmesg that the firmware has indeed been successfully loaded.
*   Bonus points: disable the module loading altogether in **Enable loadable module support** (`CONFIG_MODULES`).

Hope this helped someone and happy hacking!
