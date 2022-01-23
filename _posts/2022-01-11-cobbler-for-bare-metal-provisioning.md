---
layout: post
title:  "Cobbler for Bare Metal Provisioning"
author: "Steve O'Neill"
---
# Before you begin

This should really be behind some kind of router. I am going to use an EdgeRouter X with no ports forwarded. The EdgeRouter will provide DHCP and DNS to its internal network of 10.0.0.0/24.

Basically following this:

[How to Install and Configure Cobbler on CentOS 7.x (linuxtechi.com)](https://www.linuxtechi.com/install-and-configure-cobbler-on-centos-7/)

Create a new CentOS VM. I created mine on a CentOS host using KVM and cockpit-machines for GUI management. Making a snapshot does not appear to be supported at the time of this writing, but it can be done by command line.

It should have bridged networking and be able to access the network. A second physical NIC on the host should be directly attached so the VM can act as a PXE server for its own internal network. 

On the VM, run these commands:

```bash
**dnf install epel-release
dnf module enable cobbler
dnf install cobbler
dnf install cobbler-web**
```

I didn’t install cobbler-web at first and was confused a little bit because checking [https://host_ip/cobbler-web](https://host_ip/cobbler-web) wasn’t showing anything. Restarting apache resolved it:

```jsx
apachectl restart
```

![Untitled](/assets/img//assets/img/Cobbler%20for%20Bare%20Metal%20provisioning%2012a525a7d9f746c99fca53754ff410e5/Untitled.png)

Cobbler uses a self-signed SSL certificate. Creds are “cobbler” and “cobbler”.

We need a few more pre-requisites:

```bash
dnf install yum-utils dnf-plugins-core debmirror pykickstart syslinux xinetd
```

Interestingly, the guides I followed didn’t make it clear that you must install and configure DHCP-Server (dhcpd) beforehand. I caught this by trying to “cobbler sync” but it was also shown during “cobbler check”. It will also complain that “named” (Bind) isn’t installed, but we will be ignoring that.

1. Set a static IP for the 2nd [physical] NIC that you associated with the VM. We are going to give it a private IP, 172.168.10.1. Use the nmtui command to do so:

![Untitled](/assets/img/Cobbler%20for%20Bare%20Metal%20provisioning%2012a525a7d9f746c99fca53754ff410e5/Untitled%201.png)

1. Install DHCP server: 

```bash
dnf install dhcp-server
```

Don’t bother editing `/etc/dhcp/dhcpd.conf`. That file is dynamically updated by Cobbler. If we were to open that, it would remind us that DHCP server settings are now controlled by Cobbler anyhow:

```bash
# ******************************************************************
# Cobbler managed dhcpd.conf file
# generated from cobbler dhcp.conf template (Sun Dec 26 20:16:44 2021)
# Do NOT make changes to /etc/dhcpd.conf. Instead, make your changes
# in /etc/cobbler/dhcp.template, as /etc/dhcpd.conf will be
# overwritten.
# ******************************************************************
```

To actually make changes to the DHCP settings, edit /etc/cobbler/dhcp.template:

```bash
subnet 172.168.10.0 netmask 255.255.255.0 {
     option routers             172.168.10.1;
     option domain-name-servers 10.0.0.1;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        172.168.10.80 172.168.10.140;
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                $next_server;
     class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
```

This file is where I am defining the internal network that our PXE server will provide. It will be 172.168.10.0/24, a subnet I chose at random (make sure you choose a subnet in the RFC1918 private range. I felt foolish after realizing that 172.168.10.1 is actually a valid PUBLIC ip!) ugh

Notice that we are letting the EdgeRouter, 10.0.0.1, do DNS for now. It’s not necessary to install Bind on the VM or the host. 

Change the configuration file at /etc/cobbler/dnsmasq.template to reflect the DHCP range. Honestly I don’t even think this is used and I might take it out of this guide.

```bash
# Cobbler generated configuration file for dnsmasq
# $date 
#

# resolve.conf .. ?
#no-poll
#enable-dbus
read-ethers
addn-hosts = /var/lib/cobbler/cobbler_hosts

dhcp-range=172.168.10.80,172.168.10.140
dhcp-option=66,$next_server
dhcp-lease-max=1000
dhcp-authoritative
dhcp-boot=pxelinux.0
dhcp-boot=net:normalarch,pxelinux.0

$insert_cobbler_system_definitions
```

Run a “cobbler sync” to update. 

If you were to plug your laptop into the secondary NIC now, you would get an address in the 172.168.10.80-140 range, as expected. But you would not get internet. By default IP forwarding is disabled in CentOS. 

Restart cobbler (systemctl cobblerd restart) and run cobbler sync again

## Configure Cobbler

Download an ISO. We will start with Ubuntu Server.

```bash
wget -P /mnt/iso http://mirror.math.princeton.edu/pub/ubuntu-iso/focal/ubuntu-20.04.3-live-server-amd64.iso
```

```bash
mkdir  /mnt/iso
mount -o loop ubuntu-20.04.3-live-server-amd64.iso /mnt/iso/
cobbler import --arch=x86_64 --path=/mnt/iso --name=Ubuntu_Server_Live
```

Verify that it came down from the command line or the web console:

```bash
[root@pxe-server iso]# cobbler distro list
   Ubuntu_Server_Live-casper-x86_64
```

You can also get a report with “cobbler distro report”

```bash
[root@pxe-server iso]# cobbler distro report
Name                           : Ubuntu_Server_Live-casper-x86_64
Architecture                   : x86_64
Automatic Installation Template Metadata : {'tree': 'http://@@http_server@@/cblr/links/Ubuntu_Server_Live-casper-x86_64'}
TFTP Boot Files                : {}
Boot loader                    : grub
Breed                          : ubuntu
Comment                        :
Fetchable Files                : {}
Initrd                         : /var/www/cobbler/distro_mirror/Ubuntu_Server_Live-x86_64/casper/initrd
Kernel                         : /var/www/cobbler/distro_mirror/Ubuntu_Server_Live-x86_64/casper/vmlinuz
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Management Classes             : []
OS Version                     : focal
Owners                         : ['admin']
Redhat Management Key          :
Remote Boot Initrd             : ~
Remote Boot Kernel             : ~
Template Files                 : {}
```

Things are going well. However, this comes up during Cobbler Check:

> 2. some network boot-loaders are missing from /var/lib/cobbler/loaders. If you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely. Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, and yaboot.
> 

Solution: 

```bash
dnf install -y grub2-efi-x64-modules efibootmgr
yum install -y grub2-efi-x64 shim-x64
/usr/share/cobbler/bin/mkgrub.sh
```

This doesn’t make the error message go away because it doesn’t copy over yaboot to /var/lib/cobbler/loaders. But it does enable EFI booting.