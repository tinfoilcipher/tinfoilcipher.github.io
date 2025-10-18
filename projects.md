---
layout: page
title: Projects
permalink: /projects/
---

[Building a Bare Metal Kubernetes Lab]({% post_url 2023-01-20-building-a-bare-metal-kubernetes-lab-part-1 %})

Setup and configuration of a bare metal Kubernetes home lab using VMWare VMs. With deep dives in to the configuration of a highly available Control Plane, PKI configuration and managing the network challenges that come with a bare metal deployment.

[Terraform Modules](https://registry.terraform.io/namespaces/tinfoilcipher)

Public Terraform Modules in the official registry for various niche use cases that aren’t well covered by existing solutions and are a little too fussy to put together manually.

[Resilient Infrastructure on a Shoestring]({% post_url 2019-11-15-resiliant-infrastructure-on-a-shoestring %})

Setup of a resilient personal infrastructure at low cost, providing Linux VM hosting on a VMWare platform, segregated networking (firewall and dot1q VLANs), redundant storage and backups.

[Simple IP/HTTP Monitoring Server](https://github.com/tinfoilcipher/simple-monitoring-tool)

A simple IP and HTTP/S monitoring solution implemented in Python and Docker for use in very small networks and homelabs without the overhead of setting up a large infrastructure. Provides uptime monitoring and email alerting on a schedule.

[Secure Home VPN]({% post_url 2019-11-15-secure-home-vpn %})

Setup of a secure VPN for personal use at no cost, leveraging a Linux Operating System and the OpenVPN Access Server platform and allowing the use of MFA against for use on Windows, Linux, Android and iOS clients.

[Lightweight Internal DNS and Certificate Authority]({% post_url 2019-11-15-bind-dns-and-openssl-certificate-authority %})

Setup of a lightweight BIND9 internal DNS server and lightweight CA leveraging a Linux platform and tinyCA2 to issue certificates in conjunction with the DNS naming issued from BIND.

[WPA-EAP TLS (802.1X) WiFi]({% post_url 2019-11-15-wpa-eap-tls-wifi %})

Setup of personal 802.1x Enterprise Grade WiFi deployment using freeradius and openssl to provide enterprise grade WiFi security to a personal deployment that is compatible with any wireless access point which supports the WPA-EAP protocol.