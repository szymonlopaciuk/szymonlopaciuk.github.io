---
layout: post
title: "Putting Linux on iBook G3 Clamshell"
date: 2020-06-09 13:29:00
---

# First Attempt

Even though I am too young to remember when the iBook came out, I think it's universally seen, together with the iMac G3, as the device that brought Apple back and made it the company it is now. The aesthetic of that device was something that was never really seen before: the computer was just fun! Even though some people prefer to refer to it as the "toilet seat" iBook, since I first heard about it I always wanted to have one -- and this year I got the first edition (PowerBook2,1) Blueberry 300MHz for Christmas! And naturally, one of my first thoughts was "I wonder how I can put Linux on it?".

It took me a few months before I got to it, but a few days ago was my first attempt. Since I wanted to spare my files on the built-in 3GB IDE hard drive, I decided I want to first install Linux on a USB stick.

I decided to try and go with Debian. In hindsight, it was probably not the best choice; I did some quick research: turns out there are not many distributions out there still supporting the 32-bit PPC architecture. In fact it seems that [the choice is between Ubuntu and Gentoo][distrowatch]. I'm not a huge fan of Canonical, so it would really be just Gentoo, or willing to go with any UNIX, a range of BSD systems.

## Preparing the installation media

Finding the suitable ISO took slightly longer that anticipated, but in the end [here][debian-iso] has the necessary files. I burnt the CD, put in the USB stick, and booted the iBook while holding <kbd>C</kbd>, and... nothing happened. And by nothing, I mean the drive started making weird noises. I tested the CD in another computer, and it seemed fine. On the other hand, I booted my iBook into Mac OS 9, put a different CD in, an it got read without any issues. So either the CD was burnt badly, but not badly enough for this to be a problem for another computer, or the drive is damaged, but not badly enough for it to be a problem with another CD.

I decided I don't really want to explore this further, possibly wasting more CDs, and went for plan B: boot directly from the USB stick I want to install the OS from. The USB stick I was using is 16GB, and so I partitioned it as follows on my modern Mac:

```sh
$ sudo diskutil partitionDisk /dev/disk2 2 APM HFS+ Restore 1G HFS+ System R
```

Which means: create an Apple Partition Map with 2 partitions on `/dev/disk2`, the first partition of size 1GB should be formatted as HFS+ with the name "Restore", and second should take the rest of the space, also be formatted as HFS+, and be called "System". I then mounted the Debian ISO and copied all files from it onto "Restore". Then I was ready to try the installation again.

## Open Firmware

In order to boot a PowerPC Macintosh from a USB stick, we need to go into the Open Firmware prompt. To do this, we boot the computer while holding <kbd>⌥</kbd><kbd>⌘</kbd><kbd>O</kbd><kbd>F</kbd> on the keyboard. For a useful guide on Open Firmware, I refer the reader to [here][openfirmware]. In order to boot from USB in Open Firmware, we first need to determine the USB device:

```
 0 > dev / ls
  ...
  ff9284a8:    /usb@18
  ff93fe10:      /disk@1
  ...
```

In this case it's `usb@18`. Then, knowing that, we check the alias of the device (Otherwise we'd have to type out the whole path later on).


```
 0 > dev / ls
  ...
  usb0          /pci@f2000000/usb@18
  usb1          /pci@f2000000/usb@19
  ...
```

We can see that `usb@18` corresponds to the alias `usb0`. The bootloader called `yaboot` is in the second partition in the folder called `install`. It's in the second partition, since the partition map counts as 'partition 1'. In order to boot the installer, we type:

```
 0 > boot usb0/disk@1:2,\install\yaboot
```

We are greeted by the bootloader screen, where we need to select the `install32` option, and afterwards the process is familiar if you have ever installed Linux elsewhere.

## Issue

Or so I thought. Turns out partitioning the install media like I did is not something you are supposed to do, as the installer got confused and had me insert the install CD into the `cdrom` drive. I tried mounting the installer medium at the necessary location manually in the console, however `mount -t hfs /dev/sdb2 /cdrom` stubbornly reported that the device did not exist, and equally the `hfsutils` package programs didn't seem to be available on the install console. As I couldn't figure out how to fix this, I was faced with two options:

1. perform a network install, or
2. install the OS onto a separate USB stick on a PPC machine with 2 USB ports (PowerBook4,1).

I sort of did both approaches in parallel: as I started with option 2 I realised that I forgot how slow USB 1 really is. And during the countless hours it took to install Linux that way, I resorted to figuring out option 2. The only hiccup I had with option 1 was that the installer, for some reason, 'could not inform the kernel of a change' after I partitioned the target USB stick. A reboot fixed that.

## Booting from network

In order to set up Netboot, I followed [this tutorial][netboot] to do it on a modern Mac, and I will try to summarise the steps here. When using Linux, [these instructions][netboot-linux] can be helpful too.

First I connected the iBook and the modern Mac together with an ethernet cable. Then, on the modern Mac (which we will call the server) I went into 'Network Preferences' and set up Ethernet manually with the following settings:

- IP address: `192.168.1.1`,
- subnet mask: `255.255.255.0`,
- and the rest blank.

The next step is changing the configuration of the built-in DHCP server. To do that, I edited the following system file: `/etc/bootpd.plist`. If you are repeating my steps, make sure that the 'Internet Sharing' in 'Sharing' preferences is switched off -- switching that preference on or off will overwrite/delete the file! I put the following configuration in in `/etc/bootpd.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>dhcp_enabled</key>
	<string>en0</string>
	<key>Subnets</key>
	<array>
		<dict>
			<key>name</key>
			<string>192.168.1</string>
			<key>net_mask</key>
			<string>255.255.255.0</string>
			<key>net_address</key>
			<string>192.168.1.0</string>
			<key>net_range</key>
			<array>
				<string>192.168.1.2</string>
				<string>192.168.1.254</string>
			</array>
			<key>allocate</key>
			<true/>
			<key>dhcp_option_66</key>
			<string>192.168.1.1</string>
			<key>dhcp_option_67</key>
			<data>XHlhYm9vdAA=</data>
		</dict>
	</array>
</dict>
</plist>
```

Note that `XHlhYm9vdAA=` stands for `\yaboot`, the bootloader filename, null-terminated and base64-encoded.

Next, I prepared the TFTP server. The root of the file server is at `/private/tftpboot`, and there I needed to place the bootloader and all the necessary files that it needs. For Debian I found everything I needed [under this link][debian-netboot-files]. I `wget`-ed them all and placed them in the above-mentioned directory.

Finally, I enabled both the DHCP and TFTP servers with the following commands:

```sh
$ sudo launchctl load -w /System/Library/LaunchDaemons/bootps.plist
$ sudo launchctl load -w /System/Library/LaunchDaemons/tftp.plist
```

I boot up the iBook into Open Firmware, and typed the following command to tell it to boot from network:

```
 0 > boot enet:0,\yaboot
```

I had to wait a bit, and then screen changed to black and started throwing following errors:

```
CLIENT: 000a27aa04ae 192.168.1.2
SERVER: ffffffffffff 192.168.1.1 My-Macbook.local
Transfer FILE: 01-00-0a-27-aa-04-ae
TFTP ERROR response 2 Access violationError, can't read config file
```

At first I though that I misconfigured something. On the other hand the messages were coming from yaboot, so the bootloader must have been loaded at least somewhat correctly. After a long wait of many messages as above (with different, shorter and shorter file names), it got to `FILE: yaboot.conf`, and continued successfully. Turns out, the bootloader must be polling different filenames (more machine-specific, as above looks like a MAC address) before it gets to the general config, and since the connection is awfully slow (or rather has terrible latency, for some reason) it took its time to get there. In order to speed things up for the next time, I copied `yaboot.conf` as `01-00-0a-27-aa-04-ae`.

The installer then started successfully. With the configuration of the DHCP I had, the iBook did not have access to the wider internet, and as such I would have had to go through the installation without it. However, at that point the USB installation on the other machine had finished.

It finished with an error however: the bootloader could not be installed. Thankfully, installing it by hand is relatively easy, as we only need the `yaboot` file and a config file, `yaboot.conf`, for it. I prepared the files, copied `yaboot` out of the Debian ISO and created a `yaboot.conf` file with the following content:

```sh
# Global variables
boot=/dev/sdb3
device=usb0/disk@1:
partition=3
timeout=100
root=/dev/sdb3
read-only
default=linux
message=usb0/disk@1:2,/boot.msg

# Image labels
image=/boot/vmlinux
label=linux-cli
initrd=/boot/initrd.img
append="3"

image=/boot/vmlinux
label=linux
initrd=/boot/initrd.img
append="desktop=xfce ---" 
```

I also created an additional file, `boot.msg`, containing a welcome message.

The boot partition has the `Apple_Bootstrap` label in `APM`, however it is formatted as `HFS` (Mac OS Standard). This partition wouldn't mount on my modern Mac anymore, however [`hfsutils`][hfsutils] came to the rescue. I leave the installation of these to the reader, I had them available on my system from my previous work with [Retro68][retro68]. My USB stick was still `/dev/disk2` and so I typed:

```sh
$ sudo hmount /dev/disk2s2
$ sudo hcopy yaboot :yaboot
$ sudo hcopy yaboot.conf :yaboot.conf
$ sudo hcopy boot.msg :boot.msg
$ sudo humount
```

I plugged the USB stick back into the iBook, booted into Open Firmware and typed `boot usb0/disk@1:2,\yaboot` to boot my new Linux system, and it worked!

Side note, if you attempt the same yourself and get the following error while booting, check that you specified `initrd.img` file correctly in `yaboot.conf`:

```
Kernel Panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

This happened to me, as I got the location of that file wrong.

## Hindsight

The USB 1 port is unimaginably slow -- running an operating system from it was the initial bad idea. The boot time is in minutes, and almost every click of the mouse in the GUI (XFCE works) takes ages to complete. However, once things get loaded into memory the GUI feels quite quick, considering the machine is now 20 years old and Debian Jessie is still supported until the end of the month! To fix this, and hopefully to increase the longevity of this machine I ordered a Compact Flash to IDE converter and a 4GB CF card with which I intend to replace the internal hard disk. From there I plan to dual-boot Mac OS 9 and Linux (or possibly OpenBSD).

The next issue I experienced was a result of not hooking up the internet to my iBook during the installation. Due to this, the APT sources were configured to use the CD (or rather fake CD-USB) and I found it impossible to install new packages. To fix this I changed `/etc/sources.list` to the following:

```
deb [arch=powerpc] http://archive.debian.org/debian/ jessie main contrib non-free
deb-src [arch=powerpc] http://archive.debian.org/debian/ jessie main contrib non-free
```

This did the trick and I could install new software.

Finally, I think the choice of Debian was a misguided one, as I explained earlier. I think that next time I will attempt to install either Gentoo or OpenBSD, so that I can run an operating system that is still supported. Coupled with a lightweight window manager like awesome, iceWM, or for that extra retro feel Window Maker, I think the iBook can still serve as a nice machine for lightweight distraction-free working!

[distrowatch]: https://distrowatch.com/search.php?ostype=All&category=All&origin=All&basedon=All&notbasedon=None&desktop=All&architecture=powerpc&package=All&rolling=All&isosize=All&netinstall=All&language=All&defaultinit=All&status=Active#simple
[debian-iso]: https://cdimage.debian.org/cdimage/archive/8.10.0/powerpc/iso-cd/
[openfirmware]: https://sites.google.com/site/shawnhcorey/howto-boot-apple-powerpcs-from-a-usb-drive-in-open-firmware
[netboot]: http://hints.macworld.com/article.php?story=20130625164022823
[netboot-linux]: https://www.debian.org/releases/etch/powerpc/ch04s06.html.en
[debian-netboot-files]: http://archive.debian.org/debian/dists/jessie/main/installer-powerpc/current/images/powerpc/netboot/
[hfsutils]: https://www.mars.org/home/rob/proj/hfs/
[retro68]: https://github.com/autc04/Retro68