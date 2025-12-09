---
title: "Resilient Infrastructure On a Shoestring"
layout: post
date: 2019-11-15
categories: 
  - "linux"
  - "projects"
tags: 
  - "linux"
  - "networking"
  - "projects"
  - "sysadmin"
  - "vmware"
---

My personal infrastructure has gone through a number of iterations. Starting as a 450mhz Pentium 3 Ubuntu 7.04 server running SMB on a single 5400 RPM IDE disk cobbled together through a BT home hub and some cheap megabit switches, it later became an Ubuntu 14.06 host on a laptop with a broken screen and gigabit switches, then a Pentium 4 desktop and then a lightweight Gigabyte Brix mini-PC before I decided to get serious with the entire thing.

All of these iterations were basically functional but didn't provide much room to experiment. After suffering a catastrophic data loss when the drive controller blew on a disk, I set out to build something resilient, secure and functional, as well as a way learn about as many of the technologies that I was working with at the time and as ever, do the whole thing at the lowest possible cost.

## The Hardware

**[Host](https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c03793258)**: HP G8 MicroServer, 2.3Ghz Dual Core Xeon E3-1220Lv2, 16GB RAM DDR3 ECC  
**Host Disks**: 4 x 2TB WD HDDs, 1\* 500GB SanDisk SSD, 16GB SanDisk SD Card  
**[RAID Controller](https://support.hpe.com/hpsc/doc/public/display?docId=emr_na-c01677092):** HP P410i SmartArray SAS  
**[Firewall](https://www.juniper.net/documentation/en_US/release-independent/junos/information-products/pathway-pages/hardware/srx100/srx100-index.html)**: Juniper SRX100B  
**Managed Layer 2 Switches**: TP-Link [**TL-10116DE**](https://www.tp-link.com/uk/business-networking/easy-smart-switch/tl-sg1016de/) and **[TL-108SGE](https://www.tp-link.com/uk/business-networking/unmanaged-switch/tl-sg108s/)**  
**[Modem](https://www.tp-link.com/uk/home-networking/dsl-modem-router/td-w9980/)**: TP-Link W9980  
**[WiFi Access Point](https://dl.ubnt.com/guides/UniFi/UniFi_AP-AC_QSG.pdf)**: Ubiquiti UniFi AP-AC  
[**Backup Storage**](https://www.netgear.com/support/product/RND2000v2_\(ReadyNAS_Duo_v2\).aspx): Netgear ReadyNAS Duo2 (w/ 2 \* 2TB WD HDDs)

The hardware provides a solid base to configure a highly available and resilient infrastructure (though its low power and low RAM hardly makes it enterprise-grade), it serves it's functions with little difficulty.

Almost all of the hardware was purchased second hand, some of it was donated, some was bought new. Buying it all now better options are available but buying this same hardware at the time of writing (late 2019) can be purchased for around £600, probably significantly less if you care to be patient.

## The Host

I picked up the host for £140 on eBay and spent another £10 on a missing drive caddy, you can pick them up for around around £100-500 depending on how lucky you get in varying states and with hard drives included or not. I also purchased an aftermarket CPU, an Intel Xeon E3 for £45 to replace the Celeron that came installed and an HP P410i RAID Controller with a battery backup to replace the embedded disk controller.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09.png)

The hypervisor OS I settled on was VMware ESXi 6.5 (Free Edition), being the final version to support my CPU.

The server has 4 bays for 3.5" HDDs at a maximum capacity of 2TB each, as a single SAS connector is provided for all disks, this allows for a connection to the RAID controller and a configuration at the hardware level of RAID10 (don't be suckered in by JBOD just because it gets you more space, **this provides no redundancy what so ever**). The RAID10 configuration provides just under 4TB volume. A bigger yield can be obtained in RAID5, but this also presents a limitation in disk throughput, and presents a bigger overhead for recovery time in the event of a disk failure, as well as the requirement for a bigger backup volume, both of which are requirements I eventually decided against after a previous failure. The disks I used can be obtained for around £35 each if you shop around.

Finally the memory is **very** specific, the system supports a maximum of 16GB of PC3-12800E DDR3 ECC **unbuffered** RAM, the big CPU caching of the Xeon E3 means that the speed of this RAM can be pushed up to 1600 MHz from the 1333 MHz allowed by the Celeron. Getting just the right memory set me back another £45, if you’re going to do this don’t be suckered in by what looks like by a deal and buy **Buffered** RAM, the board will fail to POST and you’ll need to buy more.

## Installing the Hardware

Straight out of the gate I ran in to problems, the G8 doesn’t actually support the use of an internal SSD so I had to make some small hardware hacks to make that possible, specifically re-purposing the power rail and data bus intended for the optical internal Optical Disk Drive (ODD) to serve our SSD instead. Compounding problems I found that the only connector for the ODD is 4-pin Berg Connector (better recognised by people my age as a Floppy Disk Drive Connector). We can get an converter for about £3 on eBay and it doesn’t need to be anything flash. We’ll also need a standard SATA data connector, nothing flash again, the bus is only a maximum of 3mb/ps so don’t go and buy anything special, any spare cable should do the job:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

Once we’re wired in, we can get the SSD and the RAID Battery installed, these obviously were never meant to live anywhere in this server, so we’ll need to secure them in the void in the top of the chassis with the space age technology of some electrical tape. We’ll be running the blue SATA data cable and the black RAID battery cable down through the gaps in the case down towards the backplane:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.jpg)

The CPU is installation is nothing to shout about, a standard Intel LGA installation, drop the CPU on, apply some thermal paste and replace the fairly generous heat sink. The RAID card takes up the single 16 lane PCIe socket but does work without any additional driver installations (because this isn’t 2003):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04.png)

## Getting the Boot Working

The G8 comes with a built in SD Card slot which can be used as a boot device, in the BIOS this is part of the USB bus, so when setting the boot priority this is somewhat unusually set as **USB Key (C:\\)**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06.jpg)

There’s also some very unusual oddities regarding the built in disk controller; the **SmartArray B120i Embedded RAID Controller** is a _Fake RAID_ _Controller_ and not worth the silicon it's printed on.

Since I want to use the external P410i dedicated Raid Controller, it would seem safe to disabled the B120i embedded controller, however doing so also **silently disables the SATA AHCI bus** meaning that our SSD data port will stop working (don’t you love a good mystery setup, lost an hour working that out).

In the BIOS **System Settings** we can configure the priority of the **USB Key** boot (to look for an External USB Device, Internal USB Device, SD Card or all of the above in a specific order), since we need to get ESXi on to the SD card I’ll boot from an external USB stick first, after some more trial and error this **only works from the front, top USB port** and I didn’t find that documented anywhere.

Now knowing all this, we can finally get to the VMware ESXi installer and install the OS on to the SD card:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.jpg">
  <figcaption>A not at all shoddy picture of a screen, because iLO wasn’t set up yet</figcaption>
</figure>

## Boot and Configuration

ESXi installed without any any additional tweaking, setting up datastores and the vNetwork and management network was also painless. After a reboot, vSphere loaded with a known bug in 6.5 where the error “**An error occurred, please try again”. The problem appears both with the new VM and with existing ones**” appears every time you log in or refresh using either Chrome or Firefox (Chromium and IE still work). This bug can be resolved with a patch located [here](https://flings.vmware.com/esxi-embedded-host-client/bugs/677).

To install, first enable SSH on the ESXi Host and follow the below commands:

```
#--Copy patch to the /tmp dir on the ESXi Host (from a Linux Shell), on Windows use WinSCP
scp esxui-signed-12086396.vib root@<esxi_host_ip_address>:/tmp

#--Connect to ESXi Host
ssh <esxi_host_ip_address>

#--Install patch, BE SURE TO USE FULL PATH E.G. /tmp/esxui-signed-12086396.vib
esxcli software vib install -v /tmp/esxui-signed-12086396.vib
```

The hardware RAID was detected by VMware without issue, as was the internal SSD. This then allowed for the creation of two datastores, one on each volume:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07.png)

## File Server

We now have a working hypervisor and can start to install VMs. The first of which is a file and print server which offers out data using the SMB protocol (via [**Samba**](https://www.samba.org/)) and runs the [**CUPS**](http://www.cups.org/) print server. Given the large data storage requirement, the entire RAID datastore is attached to the VM as a second disk and added to the **[fstab](https://help.ubuntu.com/community/Fstab)** to ensure that it is visible at the OS level.

The Operating System for the VMs was my mainstay of Ubuntu 18.04 headless, being the best trade off of lightweight, featured and supported as well as having more experience with Debian variants than RH variants, Debian tends to suffer more driver issues so it was the obvious choice.

## NAS Storage and Backups

The NAS device is used for backups and offers a 4TB volume made up of two 2TB disks in a RAID0 set. This means that the backups volume itself is not redundant.

I wrestled with a few backup options. Initially I used several custom [rsync](https://rsync.samba.org/) scripts which were robust, but I found a number of issues and maintaining the logs was too much overhead. Following this I considered switching to Bacula or NAKIVO but they were both too enterprise ready and more than I needed.

Eventually I settled on running rsync in daemon mode on my file server and using the native backup functionality to consume the data on the file server via incremental daily backups.

## Networking

The LAN is served by 1GBps interfaces on all LAN devices with the exception of the Juniper firewall which only has 100MBps interfaces. This presents an issue for accessing the internet...but since I live in the middle of nowhere and my internet access is slower that 100MBps it's not really a big problem!

The modem is a TP-Link W9980 which is sold as a VDSL2 WiFi Router, however I'm only interested in its modem functionality. Since I use a non-BT ISP authentication is provided by MER (MAC Encapsulation Routing) which requires the use of a custom firmware in order to authenticate with the ISP. These can be obtained for around £20 now but the firmware doesn't seem as easy to find, I wouldn't recommend it!

The managed switches can be obtained for around £40 for the 10116 model and £20 for 108 model second hand depending on the day. These switches are located on different floors of the house to server different devices and are connected via a VLAN trunk.

Connected to a port on the modem is one of the managed switches (108). This connection is dedicated to a single transport VLAN which takes traffic to the firewalls untrust (outside) interface. On the inside of the firewall, traffic is segregated in to different subnets, each serving a different VLAN.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-1-1024x768.jpg">
  <figcaption>Network and backup, on the cheap</figcaption>
</figure>

The WiFi Access point is then connected to a switch with ingress allowed for multiple VLANs, allowing for the segregation of multiple wireless networks. The WiFi AP can be picked up for around £40-50 though it is pretty out of date now.

The managed switches support traditional 802.1Q VLANs, and also a very unusual (seemingly TP-Link only method of VLANs that doesn't seem to exist in any other implementation which is awful and doesn't serve any purpose I can work out).

By default, the VLAN tag of 1 (DEFAULT) cannot be removed unless the firmware is upgraded to the latest, this of course is a terrible practice, so get the firmware upgraded as soon as they're out the box.
