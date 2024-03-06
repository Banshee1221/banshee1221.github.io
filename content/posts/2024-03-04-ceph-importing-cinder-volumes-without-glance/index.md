---
cover:
  image: "posts/2024-03-04-ceph-importing-cinder-volumes-without-glance/banner.png"
  relative: false
  alt: "Ubuntu Login Prompt That Says Login Failed."
  #caption: "I'm sorry Dave, I'm afraid I can't do that."
author: "Eugene de Beste"
title: "Recovering Cloud Virtual Machine Access without GRUB (QEMU/OpenStack)"
date: "2024-03-04"
description: It's possible to directly manipulate Ceph to speed up the importing of Cinder volumes into OpenStack, rather than using Glance to fist upload an image to and then convert to a volume.
categories:
  - Technology
tags:
  - Cinder
  - Volumes
  - Cloud Images
  - VM
  - OpenStack
  - QEMU
  - KVM
  - Cloud
  - System Administration
  - Virtualization
  - Troubleshooting

showtoc: false
draft: true
---

# Cc