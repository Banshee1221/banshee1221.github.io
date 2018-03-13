---
layout: post
title: Removing CephFS from a Ceph Cluster (Luminous)
excerpt_separator:  <!--more-->
categories:
    - technology
tags:
    - ceph
    - post
    - storage
    - data
    - cephfs
    - metadata
---

While upgrading the packages for the Ceph cluster at [SANBI](https://www.sanbi.ac.za), I encountered an issue where the Ceph MDS daemon was causing the CephFS filesystem to become unresponsive and stuck in the `active(laggy)` state. I decided to strip down the CephFS deployment and reinstall it, since the existing one was for testing (set up before my time) and I wanted to do the process of setting it up from scratch.

It was surprisingly difficult to find a simple process for removing an MDS, but after I did some digging I ended up using the following:

```
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