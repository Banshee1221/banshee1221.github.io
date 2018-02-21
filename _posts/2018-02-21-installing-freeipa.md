---
layout: post
title: Installing FreeIPA on an Ubuntu Environment
excerpt_separator:  <!--more-->
---

At SANBI we've been using an old combination of OpenLDAP + Kerberos and nsswitch to provide LDAP with NFS directories for user accounts for our virtual machines and HPC cluster. This was originally put in place to make authentication into machines easier and to allow users to access and use the cluster without manual setup of directories and user accounts. Over time this set-up has grown to be messy and more effort to maintain than worth while.
<!--more-->

This prompted the decision to look at alternatives. Enter FreeIPA. The main attraction to using FreeIPA is that it is much easier to use from a management and maintenance point of view. This, coupled with the fact that it's a lot more self-contained than the original set-up prompted me to try to play around with it.

The blog post details the steps followed to set up the FreeIPA environment on servers and clients.

## Server

I created virtual machine on the same host that runs the old authentication services, for the sake of keeping sanity. This system will be replaced with an OpenStack deployment that will eventually be used for systems services as well as research work at some future point (blog post on that later). The virtual machine was configured in the following way:

- Hostname: `freeipa.sanbi.ac.za`
- IP address: `192.168.2.220`
- 1vCPU / 1GB RAM / 50GB disk

I started with a system update. The usual will suffice:
```bash
sudo apt update && sudo apt dist-upgrade -y
```

Once done it's time to install the FreeIPA stuff. This can be done through the following:
```bash
sudo apt-get install freeipa-server -y
```

This installed around 550MB worth of files + dependencies.

### Installer configuration

![Kerb auth screen](/assets/images/server_install_1.png){:class="img-responsive"}{:style="width: 39%; float: right; padding-left: 30px; margin-top: 12px; padding-bottom: 0px;"} During the install you'll be prompted with some configuration UIs. The first one you see will be to configure the Kerberos Realm configuration. Here you'll enter the name you want the realm to be. The default is often your DNS domain, but in uppercase. I chose SANBI.AC.ZA.

The next prompt will be for a kerberos server(s), I for the initial set-up I used the FreeIPA (`freeipa.sanbi.ac.za`) server as the Kerberos server as well. The prompt after that asks which server acts as the administrative server, which would be freeipa.sanbi.ac.za. Finally, on the krb5-admin-server prompt, press OK.

### Post-install configuration

Once the apt installer is complete, run the following command:

```bash
ipa-server-install
```

Here you'll be presented with a bunch of configuration options. Here's a list of the options I chose:

- `Do you want to configure integrated DNS (BIND)? [no]:` _no_
  - We have an existing DNS server at SANBI. If you need one to be set up and managed by FreeIPA then select `yes`.
- `Server host name [freeipa.sanbi.ac.za]:` _freeipa.sanbi.ac.za_
  - This is the hostname of the freeipa server for web purposes.
- `Please confirm the domain name [sanbi.ac.za]:` _sanbi.ac.za_
  - This is the name of the domain for your DNS setup.
- `Please provide a realm name [SANBI.AC.ZA]:` _SANBI.AC.ZA_
  - This is the name of the Kerberos realm for your IPA installation.

After this you'll be asked to provide a password for the Directory Manager and the IPA admin. Once done, you need to specify yes to completing the configuration.

This setup process will take some time and may fail after the `[25/28]: migrating certificate profiles to LDAP line with the following error:`

<div class="warn">
[error] NetworkError: cannot connect to 'https://freeipa.sanbi.ac.za:8443/ca/rest/account/login': Could not connect to freeipa.sanbi.ac.za using any address: (PR_ADDRESS_NOT_SUPPORTED_ERROR) Network address type not supported.
<br />
ipa.ipapython.install.cli.install_tool(Server): ERROR cannot connect to 'https://freeipa.sanbi.ac.za:8443/ca/rest/account/login': Could not connect to freeipa.sanbi.ac.za using any address: (PR_ADDRESS_NOT_SUPPORTED_ERROR) Network address type not supported.
<br />
ipa.ipapython.install.cli.install_tool(Server): ERROR The ipa-server-install command failed. See /var/log/ipaserver-install.log for more information
</div>

To fix this, you have to edit the Python installer scripts to add a slight delay to two of the functions.

Edit the `/usr/lib/python2.7/dist-packages/ipaserver/install/cainstance.py` file.

We need to add a sleeper function to two functions in the codebase. The issue is that there is a timeout trying to connect to the FreeIPA API as it is still busy starting up when the request is made. Adding the sleep function will allow enough time for the API to start up.

```python
from __future__ import print_function
...
import time
...
 
...
def __import_ca_chain(self):
    time.sleep(10)
...
 
...
def migrate_profiles_to_ldap():
   time.sleep(10)
...
```
In words, all you're doing is adding an import for the time function and then adding the time function to the `__import_ca_chain` and `migrate_profiles_to_ldap` functions. Once done:

```bash
ipa-server-install --uninstall && ipa-server-install
```

The installation of the server should now be complete. To confirm, run kinit admin and enter the password for admin, it should return no error.

You now now navigate to `https://freeipa.sanbi.ac.za` and log in with the kerberos admin user and pass.
