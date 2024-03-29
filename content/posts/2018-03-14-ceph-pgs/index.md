---
author: "Eugene de Beste"
title: "Understanding Ceph Placement Groups (TOO_MANY_PGS)"
date: "2018-03-14"
description: "This post details how to completely CephFS instances and MDS's from a Luminous cluster."
aliases:
    - "/understanding-ceph-pgs"
categories:
    - Technology
tags:
    - Ceph
    - Storage
    - Data
    - Ceph PGs
    - PGs
    - Placement Groups
---

## The Issue
My first foray into Ceph was at the end of last year. We had a small 72TB cluster that was split across 2 OSD nodes. I was tasked to upgrade the Ceph release running on the cluster from Jewel to Luminous, so that we could try out the new [Bluestore storage backend](https://ceph.com/community/new-luminous-bluestore/), and add two more OSD nodes to the cluster which brought us up to a humble 183TB.

After the upgrade was complete, I noticed the [Ceph dashboard](https://ceph.com/community/new-luminous-dashboard/) and `Ceph -s` command state the following warning:

```text
Health check update: too many PGs per OSD (232 > max 200) (TOO_MANY_PGS) 
```

At first I figured that it was probably due to a default setting in the Ceph configuration which was not changed relative to the growth that was expected for our cluster when it was initially set up. When I had a look at the `/etc/ceph/ceph.conf` file, I noticed two configurations:

```ini
osd_pool_default_size = 2
osd_pool_default_min_size = 2
osd_pool_default_pg_num = 1024
osd_pool_default_pgp_num = 1024
```

This corresponded with what I was seeing on the dashboard (or `ceph -s`):

![Ceph Pools](ceph_pools_pgs.png)

As you can see here, each of the pools that we have are assinged 1024 placement groups. 

According to the Ceph documentation, you can use the calculation `PGs =  (number_of_osds * 100) / replica count` to calculate the number of placement groups for a pool and round that to the nearest power of 2. For our environment, we have `44 osds` and `2 replicas`, so `(44 * 100) / 2 = 2200`. Rounding that up gives us `4096`.

So according to that calculation we should have `4096` placement groups per pool! That means that I should be way under the max with my `1024` per pool. But if that's the case, why is Ceph complaining?

---

## The Solution
On Luminous, using the command `ceph osd df tree` will yield the following:

```text
ID CLASS WEIGHT    REWEIGHT SIZE   USE    AVAIL  %USE  VAR  PGS TYPE NAME          
-1       183.79298        -   183T 68124G   117T 36.20 1.00   - root default       
-2        36.39600        - 37269G 14740G 22528G 39.55 1.09   -     host ceph-osd1 
 0   hdd   3.64000  1.00000  3726G  1570G  2156G 42.14 1.16 224         osd.0      
 1   hdd   3.64000  1.00000  3726G  1196G  2530G 32.11 0.89 211         osd.1      
...
 7   hdd   3.64000  1.00000  3726G  1726G  2000G 46.32 1.28 225         osd.7      
 8   hdd   3.64000  1.00000  3726G  1430G  2296G 38.37 1.06 227         osd.8      
 9   hdd   3.64000  1.00000  3726G  1759G  1967G 47.20 1.30 209         osd.9      
-3        36.39600        - 37269G 14512G 22757G 38.94 1.08   -     host ceph-osd2 
10   hdd   3.64000  1.00000  3726G  1532G  2194G 41.13 1.14 227         osd.10     
...
```
As you can see from the above, each of the OSDs have a varying amount of placement groups (`PGs`) assigned to them. This configuration was fine for previous releases of Ceph (_even though they actually recommend you don't have more than 100 PGs per OSD_), but it was [reduced to 200 per OSD in Luminous](http://docs.ceph.com/docs/master/release-notes/#v12-2-1-luminous). This is what is causing the error. To alleviate this, you can simply add the following two lines:

```ini
mon_max_pg_per_osd = 300
osd_max_pg_per_osd_hard_ratio = 1.2
```

to the `[general]` section of your ceph configuration file and push that to all the appropriate machines, then restart all the `mgr` and `mon` daemons in the cluster.

---

## The Explanation

### Resources

While the above should get rid of the warning, it's good to understand why it is actually not recommended to run more than 100 PGs per OSD. The [Ceph documentation](http://docs.ceph.com/docs/master/rados/operations/placement-groups/) on this does an ok job of explaining and I will try to expand on this. From their example:

> For instance, a cluster of 10 pools each with 512 placement groups on ten OSDs is a total of 5,120 placement groups spread over ten OSDs, that is 512 placement groups per OSD. That does not use too many resources. However, if 1,000 pools were created with 512 placement groups each, the OSDs will handle ~50,000 placement groups each and it would require significantly more resources and time for peering.

The above example is saying that you have:

- 10 pools
- 10 OSDs
- 512 PGs per pool

10 pools x 512 PGs = 5120 PGs spread over 10 OSDs, which is 512 per OSD. This is already over the recommended PGs per OSD. Once you start adding more pools, you quickly encounter issues with scaling. 1000 pools now require there to be 512000 PGs which is 51200 per OSD. This leads to significant increase in memory and CPU usage across the cluster as finding data becomes a more complex process. It also means that latency goes down.

According to the Ceph documentation, 100 PGs per OSD is the optimal amount to aim for. With this in mind, we can use the following calculation to work out how many PGs we actually have per OSD: `(num_of_pools * PGs_per_pool) / num_of_OSDs`. Using my configuration of `1024` per pool, we see that `(5 * 1024) / 44 = ~117`. This means that I'm supposed to have around 117 PGs per OSD. __This is where Ceph can be a little misleading.__ 

### PGs and Replication
When creating pools, Ceph takes the replication information in mind. If you have 2 replications and you create a new pool with `1024 PGs`, Ceph needs to ensure that the data is always available via one replication, so it doubles up the pool. This means that creating a pool with `1024 PGs` actually creates a pool of `2048 PGs` in my case. Plugging that into the equation, `(5 * 2048) / 44 = ~232`, which is EXACTLY the complaint that Ceph has for me:

```text
TOO_MANY_PGS: too many PGs per OSD (232 > max 200)
```

[This](https://gist.githubusercontent.com/Banshee1221/bfe99cec326bb0690d7f9919d9589c0b/raw/4484abc4683328764123806c5ca31a22852b9f63/pgs_stat.sh) lovely little script by [Laurent Barbe](http://cephnotes.ksperis.com/blog/2015/02/23/get-the-number-of-placement-groups-per-osd) will show you a summary of the total PGs per pool _AND_ per OSD. With it we can confirm that we actually have 2048 PGs per pool by running it and getting the following result:

```text
pool :	0	1	14	2	15	| SUM 
--------------------------------------------------------
osd.10	37	43	49	46	52	| 227
osd.11	50	42	40	47	44	| 223
osd.12	42	39	33	40	47	| 201
...
...
...
osd.29	39	47	40	38	52	| 216
osd.43	37	45	57	42	56	| 237
osd.44	52	71	63	45	68	| 299
--------------------------------------------------------
SUM :	2048	2048	2048	2048	2048	|

```

The only way to fix too many PGs for a pool would be to create a new pool, move the data from the old to the new and then delete the old.

I hope this clears things up for anyone struggling with Ceph PGs :)