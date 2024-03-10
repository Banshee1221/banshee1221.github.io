---
cover:
  image: "posts/2024-03-10-dynamic-raspberry-pi-provisioning/banner.jpg"
  relative: false
  alt: "Ubuntu Login Prompt That Says Login Failed."
  #caption: "I'm sorry Dave, I'm afraid I can't do that."
author: "Eugene de Beste"
title: "Automated and Dynamic Raspberry Pi Provisioning For The Lazy Homelabber"
date: "2024-03-10"
description: I'm lazy and don't like to manually re-provision SD cards or SSDs for use with my Raspberry Pi devices. I've developed an environment in which I can re-provision my Pis on demand without any physical intervention, which is useful for rapid prototyping. This blog post details my solution.
categories:
  - Technology
tags:
  - RaspberryPi
  - Homelab
  - Pi

showtoc: true
draft: true
---

Raspberry Pi's are great little devices to various purposes. I've got a couple of Pi4's for my homelab. That said, you'd probably be lying to me if you told me you were enthusiastic about the following scenario, especially if you make a lot of changes and/or try a lot of different operating systems:

1. Unplug Pi
2. Remove storage device (SD/SSD)
3. Plug storage device into PC
4. Flash OS image to device
5. Unplug device from PC
6. Plug back into Pi
7. Plug Pi back in
8. Configure Pi after it boots

I developed a solution for my homelab which allows me to re-provision my Pis on demand, without having to touch any of them. This blog post will detail my solution.


<figure>
    <img class="img-responsive" src="pi-cluster-on-switch.jpg" alt="My Raspberry Pi Collection"/>
    <figcaption style="margin-top: 0px; font-size: 13px; text-align: center"><i>My Raspberry Pi Collection</i></figcaption>
</figure>


# Requirements

## Hardware

There are a few hardware requirements for getting this going:

- A Raspberry Pi 4 (5 should work, but I don't have any).
- SD card reader.
- A USB-attached HDD/SSD (or NVMe drive if using a Pi5). I use the [Argon ONE M.2 Case](https://argon40.com/products/argon-one-m-2-case-for-raspberry-pi-4) for my Pis.
- A managed PoE+ capable switch which supports SSH connectivity. I use an old [Cisco 2960X](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-2960-x-series-switches/datasheet_c78-728232.html).
- A device to allow PoE for your Pi, such as [this splitter](https://www.amazon.com/Splitter-Standard-1000Mbps-Ethernet-TYPEC0503G/dp/B09GM8FB3X?th=1).
- Some computer to use as a server to provide network booting services.

## Software

I'm using a combination of Canonical's [MAAS](https://maas.io/) (Metal as a Service) along with a custom Python-based webserver that controls the Cisco switch to change the state of the PoE output on the ports.

# Setup

## The Pi

The Raspberry Pi4 does not ship with any embedded OS or firmware installed. It relies on a user to provision an SD card to either change booting parameters or boot to an operating system. We can take advantage of this to create a pre-boot environment that will allow for [PXE booting](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) the Pi.

### Prepare the Firmware

There is a software project that provides UEFI firmware images for the Pi to boot into. It can be found here: https://github.com/pftf/RPi4. The firmware needs to be written to an SD card for the Pi to originally boot from.

1. Before performing this process, the Pi needs to be set to boot from SD card first. If this isn't already done, an SD card needs to be prepared to flash this instruction to the Pi EEPROM.
    1. Download the Raspberry Pi Imager software here: https://www.raspberrypi.com/software/.
    2. For "**Raspberry Pi Device**" choose the Raspberry Pi you have (4 or 5).
    3. For "**Operating System** click **CHOOSE OS** -> **Misc utility images** -> **Bootloader** -> **SD Card Boot**.
    4. For "**Storage**" select your SD card and click **NEXT** and complete the flashing process.
    5. Put the SD card in the Pi and boot it once. It should have a green screen. This means it has successfully updated the EEPROM.
    6. Remove the SD card from the Pi.
2. Retrieve the [v1.34 UEFI firmware](https://github.com/pftf/RPi4/releases/download/v1.34/RPi4_UEFI_Firmware_v1.34.zip) from https://github.com/pftf/RPi4/releases. I use v1.34 instead of v1.35 due to a bug that prevents it from working consistently.
3. Follow the instruction for installing the UEFI firmware onto the SD card: https://github.com/pftf/RPi4?tab=readme-ov-file#installation.
4. After flashing the UEFI firmware, edit the `config.txt` file on the resulting SD card partition and add the following:
    ```ini
    ...
    hdmi_force_hotplug=1
    hdmi_group=1
    hdmi_mode=16
    ```
    This will enable the HDMI output on the Pi even if there was no HDMI device plugged into it at power on time.

    If any other customization is needed (e.g. additional overlays), make those now.
5. Put the SD card into the Pi and boot it. When you see the UEFI initialization screen, hit ESC to enter the setup screen. A couple of things need to be changed:
    1. Remove the 3GB RAM limit:

        **Device Manager** -> **Raspberry Pi Configuration** -> **Advanced Settings**.
    2. Ensure that network booting is set to be first in the boot order:

        **Boot Maintenance Manager** -> **Change Boot Order** and set UEFI PXEv4 to first and UEFI HTTPv4 to second. For extra measure, go to the **Delete Boot Order** menu and delete everything that isn't those two.
6. Reboot the Pi, but don't let it complete a boot process at this stage. Unplug the power and remove the SD card.

### Lock the SD card

Due to a quirk with the UEFI firmware, the boot order gets lost when a new EFI boot entry is written (this will happen on a new OS install). The SD card needs to be software locked to combat this. There is a tool hosted on Github that enables this: https://github.com/BertoldVdb/sdtool.

1. Grab the repository zip and extract it: https://github.com/BertoldVdb/sdtool/archive/refs/heads/master.zip
2. Compile the application with `make`.
3. Insert the SD card into your Linux machine and run the lock command on it:
    ```bash
    # sd card is usually at /dev/mmcblk<something>
    ./sdtool /dev/<sdcard> lock
    ```
4. Insert the SD card back into the Pi.

The Pi is now ready to go.

## MAAS

Canonical makes a tool called MAAS (Metal as a Service). This tool is a glorified wrapped around a DHCP server that allows you to do management of bare metal machines (and more). It's a simple 