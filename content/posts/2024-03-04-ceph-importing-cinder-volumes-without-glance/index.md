---
cover:
  image: "posts/2024-03-04-ceph-importing-cinder-volumes-without-glance/banner.png"
  relative: false
  alt: "Ubuntu Login Prompt That Says Login Failed."
  #caption: "I'm sorry Dave, I'm afraid I can't do that."
author: "Eugene de Beste"
title: "Accelerate Cinder Volume Imports in OpenStack By Avoiding Glance"
date: "2024-03-04"
description: While it may seem daunting, delving into the underlying tooling can often prove worthwhile for speeding up operations. When importing qcow2 or raw images to Ceph-backed Cinder, getting your hands a little dirty can significantly expedite the process.
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
---

If you've ever had to export and move volumes from one OpenStack cloud to another, you may know the following process: **_converting a Cinder volume to a Glance image to allow you to download it so that the reverse process can be applied when importing it_**. This is obviously a laborious process, especially if there are a ton of volumes to port.

I recently had to do a bulk import of more than 100 volumes in `qcow2` format for a client migrating to our cloud. The prospect of applying the above process to each of these images had me crawling in my skin. Instead, I followed this approach:

1. Download a `qcow2` image from the website provided by the client onto one of my Ceph nodes.
2. Convert the `qcow2` image to `raw`, as this is more appropriate for Ceph storage. Ceph handles many of the features that `qcow2` provides at the RBD level.
   ```bash
   qemu-img convert -pf qcow2 -O raw <volume_name>.qcow2 <volume_name>.img
   ```  
3. Get the resulting size of the raw image
   ```bash
   ls -laph <volume_name>.img
   ```
4. Create a volume with the appropriate name in OpenStack, then get the ID
   ```bash
   openstack volume create --size=<volume_size> <volume_name>
   ID=$(openstack volume show <volume_name> -c id -f value)
   ```
5. Identify the RBD volume location
   ```bash
   rbd -p <openstack_volume_pool> ls | grep $ID
   ```
   You should see something like `volume-<volume_id>`. Now the volume's location in Ceph is known.
6. Delete the empty RBD volume in Ceph and replace it with the converted raw image
   ```bash
   rbd -p <openstack_volume_pool> rm volume-<volume_id>
   rbd -p <openstack_volume_pool> import <volume_name>.img volume-<volume_id>
   ```

Once done, things can be verified by spinning up a VM and attaching the volume to said VM. The imported data should be visible to the VM. This can also be done for bootable volumes.

Wrapping this process up in some scripts can save **_hours_** of manual work! 