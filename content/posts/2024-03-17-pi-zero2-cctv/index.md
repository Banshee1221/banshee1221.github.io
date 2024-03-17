---
cover:
  image: "posts/2024-03-17-pi-zero2-cctv/banner.jpg"
  relative: false
  #alt: "Ubuntu Login Prompt That Says Login Failed."
  #caption: "I'm sorry Dave, I'm afraid I can't do that."
author: "Eugene de Beste"
title: "Wireless Pi Zero 2 Powered CCTV Monitor on the Cheap"
date: "2024-03-17"
description: Not too long ago I set up a few IP cameras around the house. I wanted a non-proprietary way to monitor the feeds while I'm in my study. This blog details my solution leveraging a Raspberry Pi Zero 2 W and an old monitor.
categories:
  - Technology
tags:
  - RaspberryPi


showtoc: true
TocOpen: true
draft: true
---

Not too long ago I set up a few IP cameras around the house. As I work mostly from home, I wanted a non-proprietary way to monitor the feeds while I'm in my study. This blog details my solution leveraging a Raspberry Pi Zero 2 W and an old monitor I had lying around.

download https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-05-28/

```
sudo su
apt update
apt install git omxplayer fbi vim -y
git clone https://github.com/Anonymousdog/displaycameras.git
cd displaycameras
chmod +x install.sh
./install.sh
```
gpu split 128

edit the screen size to 720p
```
framebuffer_width=1280
framebuffer_height=720
hdmi_force_hotplug=1
hdmi_group=2
hdmi_mode=85
```

```
reboot
```

uncomment blanking in displaycameras.conf

```
blank="true"
```

/etc/displaycameras/layout.conf.default

```
# This is the camera feed/windows layout configuration file for the
# displaycameras service.  It ONLY configures the layout and feeds for
# the cameras; the rest of the configuration is in displaycameras.conf.
# See the comments in that file for notes on configuring the below.

# This example defines seven 640x360 windows, three of which are off-screen to the right,
# through which the service rotates six camera feeds (it actually uses only six windows)
# on a 1280x720 monitor.  If this suites your needs, modify only the camera names to taste
# and feed URLs to what your cameras or NVR provides.

# Window names

# 2x2 screen (for 1280x720 screens)
windows=(upper_left upper_right lower_left lower_right)
# Make sure to account for each window above in the list below.

# Windows positions

window_positions=(
#First Row
#upper_left
# 640x360
"0 0 639 359" \
#upper_right
"640 0 1279 359" \

#Second Row
#lower_left
"0 360 639 719" \
#lower_right
"640 360 1279 719" \
)

# Camera Names

camera_names=(test1 test2 test3 test4)
# Make sure to account for each camera above in the list of feeds below.

# Camera Feeds

camera_feeds=(
"<camera_url_one>" \
"<camera_url_one>" \
"<camera_url_one>" \
"<camera_url_one>" \
)

# Are we rotating cameras through the window matrix?  Set this explicitly to
# "false" if not desired in this display layout.
rotate="false"
```

```
systemctl enable displaycameras
systemctl restart displaycameras
```