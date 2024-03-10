---
cover:
  image: "posts/2024-03-10-dynamic-raspberry-pi-provisioning/banner.jpg"
  relative: false
  alt: "Ubuntu Login Prompt That Says Login Failed."
  #caption: "I'm sorry Dave, I'm afraid I can't do that."
author: "Eugene de Beste"
title: "Automated and Dynamic Raspberry Pi Provisioning For The Lazy Homelabber"
date: "2024-03-10"
description: I'm lazy and don't like to manually reprovision SD cards or SSDs for use with my Raspberry Pi devices. I've developed an environment in which I can reprovision my Pis on demand without any physical intervention, which is useful for rapid prototyping. This blog post details my solution.
categories:
  - Technology
tags:
  - RaspberryPi
  - Homelab
  - Pi

showtoc: true
draft: true
---

Raspberry Pi's are great little devices to various purposes. I've got a couple of Pi4's for my homelab. That said, you'd probably be lying to me if you told me you were enthusiastic about the following scenario, espcially if you make a lot of changes and/or try a lot of different operating systems:

1. Unplug Pi
2. Remove storage device (SD/SSD)
3. Plug storage device into PC
4. Flash OS image to device
5. Unplug device from PC
6. Plug back into Pi
7. Plug Pi back in
8. Configure Pi after it boots

I developed a solution for my homelab which allows me to reprovision my Pis on demand, without having to touch any of them. This blog post will detail my solution.


<figure>
    <img class="img-responsive" src="pi-cluster-on-switch.jpg" alt="My Raspberry Pi Collection"/>
    <figcaption style="margin-top: 0px; font-size: 13px; text-align: center"><i>My Raspberry Pi Collection</i></figcaption>
</figure>


# Requirements

## Hardware

There are a few hardware requirements for getting this going:

- A Raspberry Pi 4 (5 should work, but I don't have any)
- An SD card reader
- A managed PoE+ capable switch (I use an old [Cisco 2960X](https://www.cisco.com/c/en/us/products/collateral/switches/catalyst-2960-x-series-switches/datasheet_c78-728232.html))
- A device to allow PoE for your Pi, such as [this splitter](https://www.amazon.com/Splitter-Standard-1000Mbps-Ethernet-TYPEC0503G/dp/B09GM8FB3X?th=1)
- Some computer to use to provide network booting services

## Software

I'm using a combination of Canonical's [MAAS](https://maas.io/) (Metal as a Service) along with a custom Python-based webserver that controls the Cisco switch to change the state of the PoE output on the ports.

# Setup

## The Pi

The Raspberry Pi4 does not ship with any embeded OS or firmware installed. It relies on a user to provision an SD card to either change booting paramters or boot to an operating system. We can take advantage of this to create a pre-boot environmnet that will allow for [PXE booting](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) the Pi.

### Prepare the Firmware

There is a software project that provides UEFI firmware images for the Pi to boot into. It can be found here: https://github.com/pftf/RPi4.