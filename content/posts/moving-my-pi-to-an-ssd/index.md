+++
title = 'Moving My Pi to an SSD'
date = 2026-01-20
categories = ['blog']
tags = ['rpi', 'raspberrypi', 'selfhosted', 'homeassistant']
summary = """After setting a project aside for 5 years, I got my RPi4 booting from an SSD and running a 64bit OS by tweaking
the `USB_MSD_STARTUP_DELAY` bootloader config parameter."""
draft = true
+++

{{< tldr >}} After setting a project aside for 5 years, I got my RPi4 booting from an SSD and running a 64bit OS by tweaking
the `USB_MSD_STARTUP_DELAY` bootloader config parameter.{{< /tldr >}}

I run Home Assistant on a Raspberry Pi 4, using Docker Compose on Raspbian.  This has worked great for me for years, and
overall been very stable, but there's been a Sword of Damocles hanging over the project: it was all running off the SD
card. Eventually the SD Card would accumulate too many writes and fail.  I had all the logging set up
to use [log2ram](https://github.com/azlux/log2ram) to minimize wear-and-tear, but this was a band-aid.

In 2020 the RPi got a firmware update that allowed booting from USB instead
of the SD card, which allows using an SSD with an external enclosure.  In 2021 I gave a shot to moving to SSD; there
were a number of guides available on how to do this, so I bought:

* A [128G SATA m.2 drive](https://www.amazon.com/dp/B079X7K6VP?th=1) (why not NVMe?  Price and it's going over USB3 anyway)
* A USB M.2 SATA enclosure off [this list of working hardware](https://jamesachambers.com/best-ssd-storage-adapters-for-raspberry-pi-4-400/)
* A powered USB3 hub
* (later, after I had trouble) A different [USB M.2 SATA](https://www.amazon.com/UGREEN-Enclosure-Aluminum-External-Tool-Free/dp/B07NPG5H83) enclosure off that list

And I gave a shot to copying the current SD card onto the SSD and booting up the Pi.  

## It didn't work

I tried this a number of times and different ways.  The SD card copied fine and if I hot-plugged the drive into the
running Pi, I was able to mount it and browse the files.  I triple checked the boot order, but if I left
the SD card in the Pi it would boot off the SD card, and if I removed the SD card and let it
try to boot...nothing.

{{< aside figureSrc="ShelfOfDespair.jpg" figureAlt="A shelf of abandoned projects labeled 'Shelf of Despair'" figureCaption=`Shelf
of Despair (credit: Nano Banana Pro)`>}} I don't have a literal
Shelf of Despair but now I think I might need to build one. Eerie lighting...maybe it'll play a dirge if you put a new
items on it...
{{</ aside >}}

At the time, I decided everything was working good 'nuf and I'd just restore from backup if the SD card crapped out. I put the
drive and enclosure up on the Shelf of Despair with other half-finished projects. 
## Whoops, I need 64 bits

So, as with many homelab projects that Just Work...I let it keep on just working and left it alone.  In 2025, though,
Home Assistant
[announced](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)
that it was dropping support for 32-bit OSes, which meant if I wanted to continue getting updates for Home Assistant I'd
need to reinstall my OS. This seemed like a good time to go ahead and move to an SSD.

I started by using RPi Imager to install Raspbian 64-bit to the same SSD with the same enclosure I was using before. I
tried to boot off this perfectly clean install...and it still didn't work.  Same problems as before. After digging
around online some more, I found that there's a relatively common problem: the SSD enclosure doesn't power up and
come online as quickly as the bootloader is expecting, and the RPi gives up looking for a USB drive.  There's a [setting](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#USB_MSD_STARTUP_DELAY)
that can fix this

> `USB_MSD_STARTUP_DELAY`
> 
> If defined, delays USB enumeration for the given timeout after the USB host controller has initialised. If a USB hard
> disk drive takes a long time to initialise and triggers USB timeouts then this delay can be used to give the driver
> additional time to initialise. It may also be necessary to increase the overall USB timeout
> (USB_MSD_DISCOVER_TIMEOUT).

To see the current settings, you can run `rpi-eeprom-config`.  I ran `sudo rpi-eeprom-config --edit` to update the
setting  according to whatever forum post I finally landed on (followed by a reboot to install the updated firmware
config) and it now looks like this:

```
[all]
BOOT_UART=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0

[all]
BOOT_ORDER=0xf41
USB_MSD_STARTUP_DELAY=20000
USB_MSD_BOOT_MAX_RETRIES=5
```

The `BOOT_ORDER` is specified [here](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#BOOT_ORDER),
and the default `0xf41` means "Try SD first, followed by USB-MSD then repeat (default if BOOT_ORDER is empty)".  The
`USB_MSD_STARTUP_DELAY` is the value I upped to 20s, and it, along with the max_retries, seems to have solved the problem for
me.  I didn't experiment further to see if a lower value would work, but it's definitely somewhere between 5s and 20s
for my combination of hardware.

I copied over the docker directory (which contained the MariaDB files and all my configs) from my backup to the new
drive, as well as a few other services, and now everything's running happily again.
