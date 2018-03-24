---
layout: post
title: DNS Woes With NGINX Reverse Proxy
excerpt_separator:  <!--more-->
description: I had some DNS resolution issues when using NGINX as a reverse proxy, which included the line "... could not be resolved (2 Server failure) ...". This post details how I solved this issue using dnsmasq as a workaround.
categories:
    - technology
tags:
    - proxy
    - reverse proxy
    - dns
    - dnsmasq
    - network
    - internet
    
---
While configuring some of the internal services that we host for external access through our NGINX proxy VM, I started noticing some strange behaviour. Every once in a little while, when requesting a page that was being passed through the proxy, the proxy server would respond with a `502 Bad Gateway`. It turns out that there were some issues with the resolver module for NGINX. I'll detail how I fixed it below.

<!--more-->
## Description
We have 2 nameservers at SANBI that handle out internal and external records. The external stuff is synced off to [Cloudfloor](https://www.mtgsy.net). The IP addresses of both are used in each of the `server` blocks under the `location` block for our services:
```Nginx
server {
    listen              80;

    server_name         <STUFF>

    access_log          /var/log/NGINX/proxy-access.log;
    error_log           /var/log/NGINX/proxy-error.log;

    location / {
        resolver <IP_1> <IP_2>;
        proxy_connect_timeout   10;
        proxy_send_timeout      15;
        proxy_read_timeout      20;
        proxy_pass http://$host$request_uri;
    }
}

```
This seemed to be a fairly normal configuration from what I'd been reading up online. However, I was seeing these `502` errors intermittently. For example, after restarting the `nginx` service the first poll to a hostname would result in a `502`, but the subsequent few would not, and then again a `502` after some time passes. I dug into the error logs of the proxy server and saw a suspicious looking line:
```Text
2018/03/24 22:42:42 [error] 3302#3302: *101 <server>.sanbi.ac.za could not be resolved (2: Server failure), client: <ip address> ...
```
You know that old haiku about DNS, right?
![A Haiku About DNS](/assets/images/dns_haiku.png){:class="img-responsive"}

I tried a couple of things, mainly with the proxy settings and specifying of upstream servers. In a nutshell, what didn't work for me was:
- Setting the resolver globally.
    - The exact same issue existed when this was done.
- Setting the IP addresses of the upstream (internal) server to connect to manually.
    - This alleviated the issue of connecting, but we don't have any assumptions about these IP addresses staying the same since they can be short lived VMs.
- Various bloody proxy module settings for NGINX.

This was clearly an issue with the way that NGINX was resolving hostnames from the nameservers that we have. It seemed that the server was failing to resolve the name on the first go and then does correctly afterwards, caching the entry (from what I understand) for the TTL by default.

## Solution
The stupid solution would be to just set `resolver <ns address> valid=10000000s;` or something dumb, but I'm looking for a proper answer to this problem. What I ended up doing was installing `dnsmasq` on the proxy server. I explicitly disabled the dhcp stuff to be safe
```Text
no-dhcp-interface=<NIC_name>
```
and specified my internal nameservers
```Text
server=<NS1_ip>
server=<NS2_ip>
...
```
I then changed the global resolver in the `/etc/nginx/nginx.conf` file to point to `127.0.0.1` and made sure that none of my `server` blocks had any residual `resolver` entries.

Once the`dnsmasq` and `nginx` services were restarted, everything worked without a hitch :)