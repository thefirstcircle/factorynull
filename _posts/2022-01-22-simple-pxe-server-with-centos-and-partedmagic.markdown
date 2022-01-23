---
layout: post
title:  "Simple PXE Server with CentOS and PartedMagic"
author: "Steve O'Neill"
---

# PXE

PartedMagic is a utility for wiping drives and recovering data. Until now, I have been using it on USB drives. However, it does support PXE booting functionality.

I have CentOS 8 Stream running on a simple OptiPlex 9020. I am going to provision a VM so I don’t have to wipe the configuration entirely if I need to start over. 

I’m giving the OptiPlex a DHCP reservation for 10.0.0.4. After doing that, I restart the host with the command `init 6` 

Now to provision a virtual machine using KVM:

First I need to check it it’s “good to go” for virtualization:

```bash
[root@af-162742 ~]# lscpu | grep Virtualization
Virtualization:      VT-x
```

Looks good. Install virt and cockpit-machines for GUI management:

```bash
dnf module install virt
dnf module install cockpit-machines
```

Now let’s enable the service:

```bash
systemctl enable libvirtd.service
systemctl start libvirtd.service
```

From this point I can manage the VM from the GUI at [https://hostname:9090/machines](https://hostname:9090/machines) (if you enabled cockpit during installation of the host OS)

![Untitled](assets/img/PXEdc5fea82969448d4b976df305ce8d049/Untitled.png)

## Enabling networking for the VM

Libvirt installed a virtual network bridge named `virbr0`:

```bash
[root@af-162742 ~]# virsh net-info default
Name:           default
UUID:           85e5bb62-a64a-45a8-b8f0-02be9435ce17
Active:         yes
Persistent:     yes
Autostart:      yes
Bridge:         virbr0
```

It provides networking to virtual machines in its own network but does not make the VMs accessible from the outside. You can see its configuration details by looking at `cat /var/lib/libvirt/dnsmasq/default.conf`. It provides IPs in the range of 192.168.122.2 to192.168.122.254

Instead, I want to make this VM addressable from the LAN by other devices. That will make it easier to SSH into and also allow me to use the EdgeRouter as a DHCP server someday.

We need to create our own network bridge:

Create a new file for the new adapter `br0`:

```bash
nano /etc/sysconfig/network-scripts/ifcfg-br0
```

Copy these contents:

```bash
STP=no
TYPE=Bridge
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=br0
UUID=d9552c99-d685-4b09-8fd2-a5990ffcbeb5
DEVICE=br0
ONBOOT=yes
IPV6_DISABLED=yes
```

And make a new configuration for eno1:

```bash
nano /etc/sysconfig/network-scripts/ifcfg-bridge-secondary-eno1
```

Use the following contents:

```bash
TYPE=Ethernet
NAME=bridge-secondary-eno1
UUID=1f927f35-92ec-4a25-a589-6b6d1e3f5055
DEVICE=eno1
ONBOOT=yes
BRIDGE=br0
```

OK, now that’s finished: 

```bash
[root@af-162742 network-scripts]# nmcli device
DEVICE  TYPE      STATE                   CONNECTION
br0     bridge    connected               br0
virbr0  bridge    connected (externally)  virbr0
eno1    ethernet  connected               bridge-secondary-eno1
lo      loopback  unmanaged               --
```

Now we can set up our VM via GUI. Just use the options to download the latest CentOS image and [optionally] run an unattended setup. Note: the password you select will be stored in plaintext on the root directory on the target machine:

![Untitled](assets/img/PXEdc5fea82969448d4b976df305ce8d049/Untitled%201.png)

Nice, it looks like it got an IP in the expected subnet:

![Untitled](assets/img/PXEdc5fea82969448d4b976df305ce8d049/Untitled%202.png)

Now to set up my PXE server on the VM.

Hang on - I’m going to take a snapshot. It looks like `cockpit-machines` isn’t capable of doing that from the front-end, so power the machine down and run the following:

```bash
virsh snapshot-create-as --domain af-162742-vm1 --name "af-162742-vm1-base"
```

Confirm the snapshot was created:

```bash
[root@af-162742 ~]# virsh snapshot-list --domain af-162742-vm1
 Name                 Creation Time               State
-----------------------------------------------------------
 af-162742-vm1-base   2021-12-29 17:19:04 -0500   shutoff
```

OK, now we can continue with setting our PXE server up.

## PXE

Nowadays there are tools like Cobbler and Foreman available for provisioning workstations and servers from bare metal. These work with PXE and give you nuanced control over which hosts get imaged with which images. Additionally, they can integrate with kickstart files or cloud-init to give you completely unattended OS installs. 

Since our objective is to have computers that need to be disposed get their drives wiped with minimal technician input, we don’t need this advanced stuff. We just need to be able to boot into PartedMagic.

Install SYSLINUX bootloaders and TFTP-server:

```bash
dnf install syslinux tftp-server
```

Now copy the bootloaders from the syslinux directory to your TFTP folder:

```bash
cp -r /usr/share/syslinux/* /var/lib/tftpboot
```

The PXE server is going to be looking for configuration in `/var/lib/tftpboot/pxelinux.cfg`. Create the directory and file but leave it empty for now:

```bash
mkdir /var/lib/tftpboot/pxelinux.cfg
touch /var/lib/tftpboot/pxelinux.cfg/default
```

## PartedMagic

I have pmagic_2021_11_17.iso, which I copied over to /mnt/iso. I’m going to mount it:

```bash
mount -o loop pmagic_2021_11_17.iso /mnt/iso/
```

PartedMagic includes a script which prepares the files for PXE booting. You can find it at `/mnt/iso/boot/pxelinux/pm2pxe.sh`. It’s write protected so we need to copy it to the user’s home directory:

```bash
cp /mnt/iso/boot/pxelinux/pm2pxe.sh ~
```

Let’s go ahead and see what it does:

```bash
[root@af-162742-vm1 pxelinux]# ./pm2pxe.sh
-bash: ./pm2pxe.sh: Permission denied
```

Interesting, it looks like we need to copy the read-only mounted files to another directory:

```bash
cp -r /mnt/iso ~
```

And adjust the permissions on that particular script (in the root/iso/boot directory):

```bash
cd /root/iso/boot/pxelinux/
chmod 777 pm2pxe.sh
```

Now, with the file hierarchy preserved, here is what that script outputs:

```bash
[root@af-162742-vm1 pxelinux]# ./pm2pxe.sh
Packing PMAGIC_2021_11_17.SQFS and friends ... 3625554 blocks

Copy the following files to your PXE server:
-- /root/iso/pmagic/bzImage
-- /root/iso/pmagic/{initrd,fu,m}.img
-- /root/iso/boot/pxelinux/pm2pxe/files.cgz
and use
-- /root/iso/boot/pxelinux/pm2pxe/stanza.txt
as the basis for your PXE configuration.
```

We will obligingly follow the directions:

```bash
mkdir /var/lib/tftpboot/pmagic
cp -r /root/iso/pmagic/bzImage /var/lib/tftpboot/pmagic
cp -r /root/iso/pmagic/{initrd,fu,m}.img /var/lib/tftpboot/pmagic
cp /root/iso/boot/pxelinux/pm2pxe/files.cgz /var/lib/tftpboot/pmagic
#copy the pmodules folder which contains the SQFS image (helpful later):
#cp -r /root/iso/pmagic/pmodules/ /var/lib/tftpboot/pmagic/
```

Let’s examine the stanza.txt file:

```bash
DEFAULT pmagic

LABEL pmagic
LINUX pmagic/bzImage
INITRD pmagic/initrd.img,pmagic/fu.img,pmagic/m.img,pmagic/files.cgz
APPEND edd=on vga=normal
```

This information will be given out with DHCP leases, telling the PXE-booting computers where to look for the boot files. 

Add the contents above to `/var/lib/tftpboot/pxelinux.cfg/default`.

## DHCP

Although the EdgeRouter is capable of PXE boot options, I am going to use a second NIC on the virtual machine so that the little “appliance” we’re building isn’t dependent on the router. It does require some extra steps. Since we made this VM addressable on the network, we can make use of another router some other time.

We are going to hand over a physical NIC completely to the VM:

![Untitled](assets/img/PXEdc5fea82969448d4b976df305ce8d049/Untitled%203.png)

`ip addr show` on the VM tells us that adapter was recognized as `enp7s0`

## DNSMASQ

This will be our DHCP server for clients on the PXE LAN. 

The adapter `enp7s0` needs a static IP of 10.0.10.1. I used the nmtui command but you can also create a new config file in . The gateway in this case is 10.0.0.1. 

Then install dnsmasq:

```bash
dnf install dnsmasq
```

Now edit the config file `/etc/dnsmasq.conf`:

First, back it up: 

```bash
cp /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
```

The file is well-commented. Adjust it as follows: 

```bash
nano /etc/dnsmasq.conf
```

```bash
interface=enp7s0,lo
#bind-interfaces
domain=internal.home.local
# DHCP range-leases
dhcp-range= enp7s0,10.0.10.30,10.0.10.254,255.255.255.0,1h
# PXE
dhcp-boot=pxelinux.0,pxeserver,10.0.10.1
# Gateway
dhcp-option=3,10.0.10.1
# DNS
dhcp-option=6,10.0.0.1
server=10.0.0.1
# Broadcast Address
dhcp-option=28,10.0.10.255
# NTP Server
dhcp-option=42,0.0.0.0

enable-tftp
tftp-root=/var/lib/tftpboot
```

No fancy options or menus here. This will be a barebones DHCP server. Notice that it’s in a different subnet. I’ve kept the EdgeRouter on DNS duty so I don’t have to configure forward zones.

```bash
systemctl start dnsmasq
systemctl enable dnsmasq
```

### HTTP

HTTP can be used to make the main file download faster. 

```bash
dnf install httpd
systemctl start httpd
systemctl enable httpd
#remove welcome.conf:
mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.orig
systemctl restart httpd

```

Default directory is /var/www/html. Put the contents of the ISO file there. I found that creating a symlink or editing the Apache config was more trouble than it is worth. Just copy them:

```bash
cp -r /root/iso /var/www/html
```

Now change the stanza by editing `/var/lib/tftpboot/pxelinux.cfg/`:

```bash
DEFAULT pmagic

label pmagic
kernel pmagic/bzImage
initrd pmagic/initrd.img,pmagic/fu.img,pmagic/m.img
append edd=on vga=normal netsrc=wget neturl="http://10.0.0.6/iso/pmagic/pmodules/" netargs="-U netboot --no-check-certificate"
```

We don’t have certificates since we are not using HTTPS, but I included the “no-check-certificates” argument in case I later implement it.

So - now we have a working PXE server. Make sure you are booting in legacy BIOS mode on the client computer.

**Create a snapshot before more configuration**

It’s probably good to take another snapshot here (remember that this is done on the VM host)

```bash
virsh snapshot-create-as --domain af-162742-vm1 --name "af-162742-vm1-basic-pxe"
```

# Next steps

**Postrouting**

On my PXE LAN, I want all my clients to be able to connect to the internet. That isn’t possible without some extra configuration. 

In CentOS, we need to enable IP Forwarding:

```bash
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/ip_forward.conf
```

Then, set firewalld rules in place:

```bash
systemctl start firewalld

# For TFTP:
firewall-cmd --zone=public --add-service=tftp --permanent
firewall-cmd --zone=public --add-service=dhcp --permanent
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 69 -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p udp --dport 69 -j ACCEPT
# For HTTP and HTTPS (for future use)
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 80 -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 -p tcp --dport 443 -j ACCEPT
```

The NAT Masquerade rule is the next important part that will allow clients to get to the internet. It requires both a “zone” and a “direct” rule specifying which network will be used (don’t worry, this is just firewalld-specific jargon; you will be OK if you simply run these commands)

The name of the adapter below - in this case enp1s0 - should match the *external* adapter in your VM (on the 10.0.0.0/24 network, in my case)

```bash
#NAT Masquerade rule (change adapter name to your external adapter)
firewall-cmd --zone=public --add-masquerade --permanent
firewall-cmd --permanent --direct --passthrough ipv4 -t nat -I POSTROUTING -o enp1s0 -j MASQUERADE -s 10.0.10.0/24
```

Now we can restart the firewall again and set it to always start on boot:

```bash
systemctl restart firewalld
systemctl enable firewalld
```

Now we have a PXE server that automatically serves PartedMagic.