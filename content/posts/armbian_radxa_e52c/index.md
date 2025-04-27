+++
title = "Unlocking the Radxa E52C: Installing a Clean Armbian image"
date = "2025-04-28T00:39:56+03:00"
author = ""
authorTwitter = "" #do not include @
tags = ["armbian", "debian", "radxa", "e52c", "network", "router"]
keywords = ["Radxa E52C","Armbian installation","Debian router","Custom router","Flash Armbian","RK3582 router","Homelab router","Single-board computer router","2.5G Ethernet router","Vendor-neutral Linux","Clean Linux installation","Bootloader flashing","`rkdeveloptool`","Maskrom mode","Device tree modification","Network configuration","DIY router","Linux networking","Armbian for Radxa","Radxa Rock 5C","How to install Armbian on Radxa E52C","Radxa E52C Debian router setup","Flashing Armbian on RK3582 board","Fixing second Ethernet port Radxa E52C Armbian","Using Armbian as a router OS","Installing vendor-neutral OS on Radxa E52C","Radxa E52C Armbian device tree","Booting Radxa E52C to Maskrom mode","Linux router","Debian","OpenWRT","Networking","Homelab","DIY networking"]
description = "Building a custom Debian router in my homelab using the Radxa E52C involved flashing a clean Armbian installation instead of the provided images. This post details the process of booting into Maskrom mode, installing a vendor-neutral OS on the RK3582 board, and making the OS recognize both Ethernet ports."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
My homelab has long been powered by Mikrotik hardware (with the exception of my WiFi access points), and I've been quite satisfied with their capabilities. Lately, however, I wanted to experiment with running a pure Debian router.

The device I chose was the [Radxa E52C](https://radxa.com/products/network-computer/e52c/), a small board computer with 2 2.5G ethernet ports, built-in EMMC for storage and a relatively powerful RK3582 chip (2x**A76** and 4x**A55** cores).

I won't cover its configuration or how it will fit in my current network setup as I'm still considering multiple options, and I hope I'll write them here one day.

Instead, I'll focus more on how I managed to flash a clean Armbian installation instead of using their provided Debian or OpenWRT image. I didn't want to spend time auditing their repos and their packages and this is why I decided to try installing a vendor-neutral image instead. Also, the absence of readily available instructions for installing official Debian on this specific board is why I decided to go with an Armbian image instead.


So, let's begin by downloading the our Armbian image.

You can download and flash the [image for the Radxa Rock 5C](https://www.armbian.com/radxa-rock-5c/), since they share a similar SOC.

You will also need the bootloader from the Radxa website, located [here](https://docs.radxa.com/en/e/e52c/download)

Finally, download or build `rkdeveloptool` from their [website](https://docs.radxa.com/en/compute-module/cm3/low-level-dev/rkdeveloptool). We need this tool to write to the device bootloader and EMMC.

The next step is to boot the board into Maskrom mode. This involves pressing and holding the **Maskrom** button (it is located inside a small hole) on the side of the device (a clip is necessary) during the power-on process. Once the device has power, you can release the button.

Then, plug a USB cable to the USB port next to the **Maskrom** button and to your computer and let's focus on the fun part.

```bash
# Check that the device is recognized
sudo ./rkdeveloptool ld

# Flash the bootloader
sudo ./rkdeveloptool db ~/Downloads/rk3588_spl_loader_v1.15.113.bin

# Flash armbian
sudo ./rkdeveloptool wl 0 ~/Downloads/Armbian_25.2.1_Rock-5c_bookworm_vendor_6.1.99_minimal.img

# Reboot the device
sudo ./rkdeveloptool rd
```

After the flashing finishes, you can use `minicom` to open a serial connection to the device. You will also need to plug a USB cable to the **DEBUG** port of the board.

Run
```bash
sudo minicom -w -t xterm -l -R UTF-8 -b 1500000 -8 -D /dev/ttyUSB0
```

You should see the board booting or the login page (depending on how fast you plugged the cable).
If you don't see anything try pressing Enter, or see if you miss the drivers for the `ch341-uart` chip that the board uses.

The board should work correctly except the obvious missing feature that I noticed immediately, the non-working **WAN** port.

To fix this, we will need to change the device tree file to point to the correct board, instead of the 5C we used earlier (probably this is because 5C has only one ethernet port)

To do this, open the `/boot/armbianEnv.txt` file with `nano`, or you can install `vim`.
Then we will need to edit the line that starts with `fdtfile`. Change the value it currently has with `rockchip/rk3588s-radxa-e52c.dtb`, so that you will end up with something like this
```
fdtfile=rockchip/rk3588s-radxa-e52c.dtb
```

Then reboot and you should have both NICs working.

With the Armbian image successfully flashed onto the Radxa E52C and the fix for the second network interface implemented, the board is now ready for its role as a Debian router. Getting a clean Armbian image up and running on the Radxa E52C certainly involved a few more steps than simply flashing a provided image.
I'm excited to experiment with different configurations and see what this little Debian machine can do, something I intend to share in a future update.
