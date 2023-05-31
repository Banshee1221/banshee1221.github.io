---
author: "Eugene de Beste"
title: "Generating Let's Encrypt Wildcard Certificates"
date: "2018-03-24"
aliases:
    - "/letsencrypt-wildcards"
categories:
    - Technology
tags:
    - Security
    - Certificates
    - SSL
    - Encryption
    - Network
    - Internet
    - Letsencrypt
---

With the recent release of [Let's Encrypt](https://letsencrypt.org/)'s ACMEv2 protocol implementation, they've gained the ability to not only supply SSL certificates for single domains, but also all subdomains. I've been interested in switching from our previous CA to Let's Encrypt when their wildcard support dropped, because it makes renewal of certificates significantly easier due to automation capabilities of the platform. This blog post describes how to generate a wildcard certificate using `Certbot`.

## Acquire Certbot
`Certbot` is the tool developed by the guys over at the [Electronic Frontier Foundation (EFF)](https://www.eff.org/) in order to simplify the lives of people using Let's Encrypt (and other ACME protocol based CAs) by automatically fetching and deploying certificates.

First, we need to get the `Certbot` executable. The latest release is available at the [EFF's website](https://dl.eff.org/certbot-auto).

```bash
wget https://dl.eff.org/certbot-auto
chmod a+x ./certbot-auto
```

## Acquire Certificate
Let's Encrypt gives you 3 ways to verify that you own the domain(s) in question: `http`, `dns` and `tls-sni` challenges. I used the `dns` challenge in my case, since [it appears that that's the only type that wildcard certificates support](https://community.letsencrypt.org/t/acme-v2-and-wildcard-certificate-support-is-live/55579).

Generate a certificate with `Certbot`:

```bash
sudo ./certbot-auto certonly \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --manual \
  --preferred-challenges dns \
  -d <your_domain_here> \
  -m <your_email_here> \
  --agree-tos \
  --no-eff-email \
  --manual-public-ip-logging-ok
```

<br />
{{< notice>}}
If you do want to share your email with the EFF, replace the `--no-eff-email` flag with `--eff-email`.
{{< /notice >}}

In my case, I used the following arguments:
```bash
sudo ./certbot-auto certonly \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --manual \
  --preferred-challenges dns \
  -d sanbi.ac.za \
  -d *.sanbi.ac.za \
  -d *.wp.sanbi.ac.za \
  -m eugene@sanbi.ac.za \
  --agree-tos \
  --no-eff-email \
  --manual-public-ip-logging-ok
```

## Add DNS Records
Once the certificate is obtained, `Certbot` will prompt the user with a message similar to the following:

```text
Performing the following challenges:
dns-01 challenge for sanbi.ac.za
-------------------------------------------------------------------------------
Please deploy a DNS TXT record under the name
_acme-challenge.sanbi.ac.za with the following value:

RjMppEEL6MImK8EB5LSadOLsrdcxNIbGxUk8rjxWd6U

Before continuing, verify the record is deployed.
-------------------------------------------------------------------------------
```

In your DNS provider configuration, you need to add a new `TXT` record name with `_acme-challenge.<your_domain_here>` and the corresponding value provided by the program. Do this for all the domains that you have specified.

If you run your own named/bind9 server, add the following line, update your serial and reload your rules:

```text
_acme-challenge.<your_domain_here>.    IN  TXT <value>
```

Before you continue with `Certbot`, check that your DNS server records have propagated using `dig` or `nslookup`. For example: `dig -t TXT _acme-challenge.sanbi.ac.za` will produce something like:

```text
; <<>> DiG 9.12.1 <<>> -t TXT _acme-challenge.sanbi.ac.za
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25805
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;_acme-challenge.sanbi.ac.za.	IN	TXT

;; ANSWER SECTION:
_acme-challenge.sanbi.ac.za. 3599 IN	TXT	"RjMppEEL6MImK8EB5LSadOLsrdcxNIbGxUk8rjxWd6U"

;; Query time: 462 msec
;; SERVER: XXX.XXX.XXX.XXX#XX(XXX.XXX.XXX.XXX)
;; WHEN: Sat Mar 24 14:02:10 SAST 2018
;; MSG SIZE  rcvd: 112
```

## Apply Certificates
Now that your certificates are generated, you can apply it to the webserver of your choice. The files will be found in `/etc/letsencrypt/live/<your_domain>/*.pem`.