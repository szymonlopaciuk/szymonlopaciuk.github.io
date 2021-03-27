---
layout: post
title: "Apple Desktop Bus Protocol"
date: 2021-03-26 18:36
---

# Backstory

When I was very young my mum had a Power Macintosh 8500, which she used for her desktop publishing work. Eventually, as with the, I believe, Quadra that preceded it, the time came that the Power Macintosh was replaced and she switched to a Power Mac G4. That was just around the time that I was old enough to start being interested in computers. My parents would sit me in front of that Power Macintosh that we still had, and let me play various games. I remember with fondness the 'Lemmings', 'Fury of the Furries', a bunch of what I now believe was Dutch educational games, and the beginnings of 'online gaming', which then was browser games running on Internet Explorer 5.

However, this is not the point of this story. The point is that the Power Macintosh came with the Apple Extended Keyboard II, which, only years later, I learnt, is still regarded as one of the best mechanical keyboards ever made. It was the Apple equivalent of IBM Model M. This keyboard has been at my parents' all these years, and even though the plastic yellowed with age, it still works with that old Power Macintosh and other old Macs.

I decided it would be nice to make it useful, and get it running again, this time with modern hardware. Once I gave it a retrobrite treatment (in my case just some hydrogen peroxide in the form of a hair bleach cream) I decided to look into my options of transforming it to USB: an obvious choice was one of the ADB to USB adapters, but as they can get quite pricey, I decided to make one myself, and maybe learn a thing or two in the process.

It is not something that hasn't been done before either. There is a variety of bigger and smaller projects accomplishing the same thing, and more, with Arduinos, Teensies, etc. I the end I decided I wanted to go the hardcore route and write the code myself. For the development platform I selected the STM32 board called the 'Blue Pill', as it was very cheap, and had many GPIO pins (which is something I am planning to use in an upcoming project).

In this post however, I want to solely focus on giving the specification of the ADB protocol. The reason for this is that a lot of the information for it I found scattered around the web, and I would have loved to find one comprehensive, but not too waffly summary of it. In the end, most of the information that was useful to me came from Apple's *Guide to the Macintosh&copy; Family Hardware*[^1], hereafter referred to as Apple's *Guide*. However even in there, there is some lack of clarity regarding how service requests are issued, for example.

For the code I refer to my GitHub project, [stm32-adb2usb](https://github.com/szymonlopaciuk/stm32-adb2usb/).

# ADB: An Introduction

Apple Desktop Bus is a serial connection that allows chaining multiple devices of different devices which can utilise a single bus, not unlike I²C. It is uniquely suited towards human input devices such as mice and keyboards, as well as tablets and trackballs, where already the design considerations behind it are such that user input is responsive and snappy. Capable of transmission speed of 10 kb/s[^2], although is reality slower because of a shared bus and polling frequency of the host.

There is one host which drives the bus and issues commands to the devices. Besides one special case, the short period when a device can assert a request signal, the devices are not allowed to use the bus unprompted. The bus when unused is pulled up to 5V high, and each of the devices has the ability to pull it low.

Key features of the protocol:

- Each device has an address which identifies it on the bus: initially this determines the type of the device, but can be changed by the host when there are multiple devices of the same time to avoid collisions.
- Each device has up to 4 so-called registers, which are 16 bits, and store data produced by the device, or which can be written to, to change the device's behaviour: device address is stored as part of register `3`.
- The host can issue 4 commands, to read (`Talk`), write (`Listen`), reset (`SendReset`), or flush (`Flush`) a device.

# Setup

The bus initialised by performing a reset signal: the host pulls the bus low for the period of 3 milliseconds or more (a summary of all the timings can be found towards the end of this post).

![ADB Reset Signal](/assets/adb-reset.svg)
*The reset signal.*

After the reset signal, it is customary for the host to wait up to one second, as some devices, notably the AEKII, are know to take a bit to reset.

Following that, the host queries the devices and reassigns their addresses to disambiguate them. This is due to a built-in collision detection. Let us assume there are two keyboards on the bus, and initially both have the address of `2` (the default address of a keyboard):

1. The host issues a `Talk` register `3` command to address `2`, and keyboard A responds first (wins the collision)
2. Keyboard B detects that it lost the collision, and immediately stops transmitting. It will also keep quiet for the next transaction issued to its address.
3. The host issues a `Listen` register `3` command to address `2` giving keyboard A a new address of, say, `8`.
4. The host issues a `Talk` register `3` command to address `2` again, and this time keyboard B responds with the contents of its register 3.
5. The host issues a `Listen` register `3` command to address `2` giving keyboard B a new address of, say, `9`.
6. The host issues a final `Talk` register `3` command to address `2` but as there are no more keyboard, there is no response. The host then moves on to querying the next address.

Keep in mind that this part is not mandatory, and as such if you are sure that you will only ever handle one device of each type it can be skipped.

After all devices are properly set up, the actual communication can commence.

![Structure of a command](/assets/adb-bits.svg)
*Bits sent by the host and a device have the structure as shown. Period of low, followed by a period of high: 65 μs and 35 μs for a `0`, and 35 μs and 65 μs for a `1`.*

# Commands

As there are four commands, which usually consist of 4 address bits, 2 command code bits, and 2 register bits. For a simple keyboard and mouse scenario, the commands `SendReset` and `Flush` do not seem necessary, however I am covering them here for completeness.

| Command    | Bits | Description |
|------------|------|-------------|
| `SendReset`  | `****0000` | Reset all the devices on the bus to their initial state. |
| `Flush`      | `AAAA0001` | Defined for each device. Usually used to clear a specific register.
| `Listen`     | `AAAA10RR` | Write data to a device register. The transaction is cancelled if there is any other signal on the bus before the data transfer starts. |
| `Talk`       | `AAAA11RR` | Request data from a device. If no new data is available the device can ignore the request, with the exception of when the command is issued to register 3. Such command must always be responded to. |

Where `AAAA` stands for the device address, `*` can be any bit, and `RR` stands for the register number.

# Polling

After all collisions are dealt with, the host begins polling devices. It starts with the default device, usually a mouse. This device is the active device, and it will be polled continuously, unless another device asserts a service request.

Each transaction is initiated by the host, which sends a command. The structure of a command is as follows:

1. `Attention` signal: the host pulls the bus low for 800 μs.
2. `Sync` signal: the bus is high for 70 μs.
3. The command is transmitted, 8 bits structured as follows: 4 bits of the device address, 2-bit command code, and 2-bit register code.
4. Stop bit of `0` is sent. During the low part of the stop bit signal any device can pull the bus low for a total of 300 μs to assert a service request (`Srq`). This is to indicate that a device is in need of service.
5. After the stop bit, the bus is kept high for 140--260 μs, the so-called 'Stop-bit-to-start-bit' time, also known as `Tlt`.
6. The queried device responds (although if no new data is available, it does not have to, depending on the register) by sending a start bit of `1`, 16 bits of the register data, followed by the stop bit of `0`.

If a device asserts an `Srq`, the host will poll the register `0` of all the devices on the bus in sequence until it finds the asserting device: that happens when no `Srq` is asserted during the transaction. This device then becomes a new active device, and will be polled continuously until another device asserts an `Srq` again. For more details on the `Srq`, [this post](https://www.bigmessowires.com/2016/03/30/understanding-the-adb-service-request-signal/) is very helpful, and provides examples of actual transactions captured using a logic analyser!

![Structure of a command](/assets/adb-command.svg)
*Structure of a command.*

![A command where an `Srq` is asserted](/assets/adb-command-srq.svg)
*Structure of a command, where a device asserted an `Srq`.*

# Registers

## Register 0

This is the main register of every device, which contains input data such as key press information for a keyboard, or mouse offset data for a mouse. Apple's *Guide* notes that it's crucial for the device asserting an `Srq` to have data in register `0`, even if data of significance is in a different register.

In further sections I will go through the specific structures of the registers for each device.

## Registers 1 and 2

These are device specific registers, and can be used for miscellaneous data.

## Register 3

This device contains device's settings and metadata. It contains the address, and a so-called handler ID, which can be changed by the host to modify devices behaviour: e.g. in the case of AEKII we change the handler ID to enable the 'Extended Keyboard Protocol' which allows for discerning left and right modifiers. The structure of the register 3 is as follows (Apple's *Guide*, p. 322):

| Bit   | Description                                     |
|:------|:------------------------------------------------|
| 15    | Reserved, must be 0                             |
| 14    | Exceptional event, device specific; 1 if unused |
| 13    | Enable `Srq`; 1 if enabled                      |
| 12    | Reserved, must be 0                             |
| 11--8 | Device address (4 bits)                         |
| 7--0  | Handler ID                                      |

# Devices

Initially device type is determined by its address. The first 8 addresses are assigned by Apple and have fixed meanings, as follows (Apple's *Guide*, p. 322):

| Address  | Description                                     |
|:---------|:------------------------------------------------|
| \$0, \$1 | Reserved                                        |
| \$2      | Encoded devices: e.g. keyboards                 |
| \$3      | Relative devices: e.g. mice, trackpads          |
| \$4      | Absolute devices: e.g. graphic tablets          |
| \$5--\$7 | Reserved                                        |
| \$8--\$F | Any other (devices can be reassigned to these)  |


## Keyboard

There are two keyboard protocols outlined by Apple: Standard and Extended. For both protocols, register 0 contains information about two key events, for each event it gives the key code and a flag for whether the key was released or pressed down. The exception to this is the power key, which takes up both bytes of the register. The structure of the register is as follows (Apple's *Guide*, p. 307):

| Bit   | Description                       |
|:------|:----------------------------------|
| 15    | First key event: 1 if released    |
| 14--8 | First key code                    |
| 7     | Second key event: 1 if released   |
| 6--0  | Second key code                   |

Extended protocol discerns between left and right modifiers, as mentioned before, and allows the host to control the LEDs on the keyboard.

Register 2 differs based on the protocol (Apple's *Guide*, pp. 307, 310):

| Bit | Description (Standard)| Description (Extended) |
|:----|:----------------------|:-----------------------|
| 15  | Reserved              | Reserved               |
| 14  | Delete (Backspace)    | Delete (Backspace)     |
| 13  | Caps Lock             | Caps Lock              |
| 12  | Reset                 | Reset                  |
| 11  | Control               | Control                |
| 10  | Shift                 | Shift                  |
| 9   | Option (Alt)          | Option (Alt)           |
| 8   | Command               | Command                |
| 7   | Reserved              | Num Lock/Clear         |
| 6   | Reserved              | Scroll Lock            |
| 5--3| Reserved              | Reserved               |
| 2   | Reserved              | Scroll Lock LED        |
| 1   | Reserved              | Caps Lock LED          |
| 0   | Reserved              | Num Lock LED           |

Bits 0-2 can be written to using the `Listen` command to alter the status of the LEDs. In all the bits one indicates that the key is released, or that the LED is off.

The list of key codes can be found [here](https://github.com/szymonlopaciuk/stm32-adb2usb/blob/main/src/keymap.h) (for an ISO layout; I have reassigned 3 F-keys to volume, but that is clearly marked), or in Apple's *Guide*, pp. 306, 308. A note regarding the layouts needs to be made here:

- on the ANSI layout the top-left <kbd>&grave;</kbd> key has the code `$32`,
- on the ANSI layout the <kbd>&#92;</kbd> key above return has the code `$2A`,
- on the ISO layout the key to the right of the left shift key, <kbd>&grave;</kbd> or <kbd>|</kbd>, has the code `$32`,
- on the ISO layout the key in the top-left corner, <kbd>§</kbd> or <kbd>¬</kbd>, has the code `$0A`,
- on the ISO layout the key to the left of the return key, <kbd>&#92;</kbd> or <kbd>#</kbd>, has the code `$2A`.

## Mouse

Similarly to the situation with the keyboard, there are two mouse protocols, standard (handler ID of `$0001`), and extended (`$0004`). For the standard protocol register 0 takes the following form (Apple's *Guide*, p. 301):

| Bit   | Description                                   |
|:------|:----------------------------------------------|
| 15    | Button status: 1 if released                  |
| 14--8 | Y axis moves (2's complement, negative up)    |
| 7     | Reserved                                      |
| 6--0  | X axis offset (2's complement, negative left) |

When using the standard protocol, the pointing device accumulates 100±10 counts per inch, however the precision can be changed to 200±10 counts per inch by setting the handler ID to `$0002`.

The extended mouse protocol is more complex, and can handle even more buttons and precision. I will skip the description here, as I have not implemented it, and do not have a supporting mouse. However, all information that seems necessary can be found [here](https://developer.apple.com/library/archive/technotes/hw/hw_01.html#Extended)[^3].


# Timings

Copied for reference directly from Apple's *Guide*:

| Parameter             | Nominal | Host                     | Device                   |
|:----------------------|:--------|:-------------------------|:-------------------------|
| Bit-cell time         | 100 μs  | ±3%                      | 3 ms minimum             |
| Bit '0' low time      | 65 μs   | 65% of bit cell time ±5% | 65% of bit cell time ±5% |
| Bit '1' low time      | 35 μs   | 35% of bit cell time ±5% | 65% of bit cell time ±5% |
| Attention (low) time  | 800 μs  | ±3%                      | N/A                      |
| Sync (high) time      | 65 μs   | ±3%                      | N/A                      |
| Stop bit low time     | 70 μs   | ±3%                      | ±30%                     |
| Global reset low time | 3 ms    | 3 ms minimum             | 3 ms minimum             |
| `Srq` low time        | 300 μs  | N/A                      | ±30%                     |
| `Tlt` (high) time     | 200 μs  | 140--260 ms              | 140--260 ms              |


[^1]: Apple Computer (1990). *Guide to the Macintosh family hardware*. 2nd ed. London, England: Addison Wesley. Available at: https://archive.org/details/apple-guide-macintosh-family-hardware.
[^2]: Wikipedia gives a figure one order of magnitude higher, although I'm not quite sure how they got there, since 1 bit takes 100 μs to send.
[^3]: Apple Computer (1994). *ADB - The Untold Story: Space Aliens Ate My Mouse*. Technical Note HW01. Archive.org mirror: https://web.archive.org/web/20210321183851/https://developer.apple.com/library/archive/technotes/hw/hw_01.html