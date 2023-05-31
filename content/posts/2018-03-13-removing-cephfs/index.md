---
author: "Eugene de Beste"
title: "Removing CephFS from a Ceph Cluster (Luminous)"
date: "2018-03-13"
description: "This post details how to completely CephFS instances and MDS's from a Luminous cluster."
aliases:
    - "/removing-cephfs"
categories:
    - Technology
tags:
    - Ceph
    - Storage
    - Data
    - CephFS
    - Metadata
---


While upgrading the packages for the Ceph cluster at [SANBI](https://www.sanbi.ac.za), I encountered an issue where the Ceph MDS daemon was causing the CephFS filesystem to become unresponsive and stuck in the `active(laggy)` state. I decided to strip down the CephFS deployment and reinstall it, since the existing one was for testing (set up before my time) and I wanted to do the process of setting it up from scratch.

It was surprisingly difficult to find a simple process for removing an MDS, but after I did some digging I ended up using the following:

```bash
systemctl stop ceph-mds.target
killall ceph-mds

ceph mds cluster_down
ceph mds fail 0

ceph fs rm <cephfs name> --yes-i-really-mean-it

ceph osd pool delete <cephfs data pool> <cephfs data pool> --yes-i-really-really-mean-it
ceph osd pool delete <cephfs metadata pool> <cephfs metadata pool> --yes-i-really-really-mean-it

rm -rf "/var/lib/ceph/mds/<cluster-metadata server>"

ceph auth del mds."$hostname"
```

Replace the things in <> with the appropraite values for your system.

Now you should be able to go ahead and reinstall CephFS on your cluster.