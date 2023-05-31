---
author: "Eugene de Beste"
title: "Installing FreeIPA 4 on an Ubuntu Environment"
date: "2018-02-22"
description: "Installing FreeIPA 4 on an Ubuntu Environment, including the server and client."
aliases:
    - "/installing-freeipa4-ubuntu-14"
categories:
    - Technology
tags:
    - First
    - FreeIPA
    - Authentication
    - Authorization
    - Federated Authentication
    - Federated
---

At SANBI we've been using an old combination of OpenLDAP + Kerberos and nsswitch to provide LDAP with NFS directories for user accounts for our virtual machines and HPC cluster. This was originally put in place to make authentication into machines easier and to allow users to access and use the cluster without manual setup of directories and user accounts. Over time this set-up has grown to be messy and more effort to maintain than worth while.

This prompted the decision to look at alternatives. Enter FreeIPA. The main attraction to using FreeIPA is that it is much easier to use from a management and maintenance point of view. This, coupled with the fact that it's a lot more self-contained than the original set-up prompted me to try to play around with it.

The blog post details the steps followed to set up the FreeIPA environment on servers and clients.

---

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

{{< figure src="server_install_1.png" class="floatright">}}

During the install you'll be prompted with some configuration UIs. The first one you see will be to configure the Kerberos Realm configuration. Here you'll enter the name you want the realm to be. The default is often your DNS domain, but in uppercase. I chose SANBI.AC.ZA.

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

{{< notice warning >}}

This setup process will take some time and may fail after the `[25/28]: migrating certificate profiles to LDAP line with the following error:`

```shell
[error] NetworkError: cannot connect to 'https://freeipa.sanbi.ac.za:8443/ca/rest/account/login': Could not connect to freeipa.sanbi.ac.za using any address: (PR_ADDRESS_NOT_SUPPORTED_ERROR) Network address type not supported.
ipa.ipapython.install.cli.install_tool(Server): ERROR cannot connect to 'https://freeipa.sanbi.ac.za:8443/ca/rest/account/login': Could not connect to freeipa.sanbi.ac.za using any address: (PR_ADDRESS_NOT_SUPPORTED_ERROR) Network address type not supported.
ipa.ipapython.install.cli.install_tool(Server): ERROR The ipa-server-install command failed. See /var/log/ipaserver-install.log for more information
```

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
{{< /notice >}}



The installation of the server should now be complete. To confirm, run kinit admin and enter the password for admin, it should return no error.

You now now navigate to `https://freeipa.sanbi.ac.za` and log in with the kerberos admin user and pass.

---

## Client

Most of the virtual machines at SANBI run some variant of Ubuntu. This ranges from a few outliers still on 12.04, some on 14.04 and most on 16.04. The machines that host these virtual machines are all CentOS 6.x based. Both the virtual machine hosts and virtual machines themselves were added to the ipa domain.

Before setting up the client, ensure that a FQDN is used as the hostname for the machine. For example, if the machine is configured as `node11.sanbi.ac.za` on DNS, ensure that the machine itself reflects this in the `/etc/hostname` and `/etc/hosts` files. This applies to both Debian and RHEL based distributions.

### CentOS 6.x

On CentOS (version 6 at least), it's fairly easy to get the machine to join the ipa domain using the `ipa-client-install`. Once the command is run, you will be prompted with the following:

- `Provide the domain name of your IPA server (ex: example.com):` _freeipa.sanbi.ac.za_
  - This is the FQDN of the freeipa server.
- `Provide your IPA server name:` _freeipa.sanbi.ac.za_
  - This is the FQDN of the freeipa server.
- `Autodiscovery of servers for failover cannot work with this configuration...` _yes_
  - This ignores looking for failovers. This will be set up at a later stage.
- `Continue to configure the system with these values? [no]:` _yes_
  - This confirms the installation.
- `User authorized to enroll computers:` _admin_
  - This is the admin (default) user for the FreeIPA installation.

Once you've entered the password, the `ipa-client-install` script will take care of configuring the machine. If all goes well, you can run the command `id admin` and you should get a result that looks something like this:

```bash
uid=10000(admin) gid=10000(admins) groups=10000(admins)
```

{{< notice >}}
`--mkhomedir` was not specified for `ipa-client-install` here because the machines running CentOS at SANBI are mostly used as service hosts, which only admins have access to.
{{< /notice >}}

### Ubuntu 14.04+

I had some issues getting FreeIPA working with Ubuntu 12.04, so I'll ignore that for the time being. Getting it working in 14.04+ is a little more involved than it is on CentOS. It seems that the `pam.d/*` files __\*sometimes\*__ don't get configured correctly for Ubuntu when using the `ipa-client-install` script. This results in `authentication failure` error messages appearing when trying to log in to the system and forced me to have to enter `single` user mode in order to restore the system to a working order.

Here's the gist: 

The installation of the FreeIPA software is done the same way as with CentOS. `apt-get install freeipa-client` will get you access to the `ipa-client-install` script and I specified the `--mkhomedir` flag with that, since a lot of the Ubuntu VMs are user-facing. The full command I used is: `ipa-client-install --domain=sanbi.ac.za --server=freeipa.sanbi.ac.za --realm=SANBI.AC.ZA -p admin --mkhomedir`. After the installation is done you can test logging in and using sudo. If all works correctly, you are done.

#### Issues

If there are login issues, you can try the following. A couple of items need to be added or changed in the `/etc/pam.d/*` directory. If these lines exist you need to make sure they look the same as the following, otherwise you can add them:

| **File**        | **Change** |
| --------------- | ------ |
| common-account  | Add: `session required pam_mkhomedir.so skel=/etc/skel/ umask=0022` |
| common-auth     | Add: `auth  [success=4 default=ignore]      pam_sss.so use_first_pass` |
| common-account  | Add: `account sufficient         pam_localuser.so`, `account [default=bad success=ok user_unknown=ignore] pam_sss.so` |
| common-password | Add: `password        sufficient      pam_sss.so` |
| common-session  | Add: `session optional                pam_sss.so` |

and in the `/usr/share/pam-configs/my_mkhomedir` file (create it if it's not there already), add the below:

```bash
Name: activate mkhomedir
Default: yes
Priority: 900
Session-Type: Additional
Session:
  required   pam_mkhomedir.so umask=0022 skel=/etc/skel
```

{{< notice >}}
If you don't want user account directories created on login OR you have a setup where user directories can't be created on setup then omit the steps pertaining to home directories, i.e. leave out `--mkhomedir` and the pam.d configurations.
{{< /notice >}}