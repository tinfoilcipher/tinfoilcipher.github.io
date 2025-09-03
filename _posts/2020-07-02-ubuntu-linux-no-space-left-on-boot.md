---
title: "Ubuntu Linux - No Space Left on /boot"
layout: post
date: 2020-07-02
categories: 
  - "linux"
tags: 
  - "linux"
  - "sysadmin"
  - "ubuntu"
---

I've encountered this issue a couple of times in the last couple of weeks and it's one that it seems unless you know the inside lore of how Linux works the actual solution isn't exactly obvious and you can easily lead you to a disaster that **seems** like it should work and can actually leave you without a bootable system. While the fix **is** technically documented the actual method is spread over a few sources and most forum posts I've seen tend to lead to slightly inaccurate solutions that don't quite understand the cause.

## The Error and The Cause

So the error that tends to lead to so many forum posts usually occurs when someone runs the dreaded **apt-get upgrade** (upgrading every package in the OS), or it can rear it's head on systems that have had a bunch of packages installed over time and eventually had several updated kernels pulled down (all those packages you just say **YES** to when you install dependencies, they all build up somewhere and your kernels end up in **/boot**).

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png">
  <figcaption>Never get's less relevant<br/>Source: https://xkcd.com/349/</figcaption>
</figure>

Usually, an error suddenly rears it's head along the lines of:

```bash
# dpkg: error processing archive /var/cache/apt/archives/linux-image-X.X.X-XXX-generic_X.X.X-XXX.XXX_arch.deb (--unpack):
# cannot copy extracted data for './boot/vmlinuz-X.X.X-XXX-generic' to '/boot/vmlinuz-X.X.X-XXX-generic.dpkg-new': failed to write (No space left on device)
# No apport report written because the error message indicates a disk full error dpkg-deb: error: subprocess paste was killed by signal (Broken pipe)

# Errors were encountered while processing:
#  /var/cache/apt/archives/linux-image-X.X.X-XXX-generic_X.X.X-XXX.XXX_arch.deb
# E: Sub-process /usr/bin/dpkg returned an error code (1)
```

In that mouthful of an error message are the killer clues "**No space left on device**" and the preceding path in **/boot**. If we take a look at the disk space using **df** we can see that the **/boot** partition is stuffed:

```bash
$ df -h
# Filesystem                          Size  Used  Avail Use%  Mounted on
# udev                                981M     0  981M    0%  /dev
# tmpfs                               201M   21M  180M   11%  /run
# /dev/mapper/test--server--vg-root    15G  3.0G   11G   22%  /
# tmpfs                              1001M  4.0K 1001M    1%  /dev/shm
# tmpfs                               5.0M     0  5.0M    0%  /run/lock
# tmpfs                              1001M     0 1001M    0%  /sys/fs/cgroup
# /dev/sda1                           472M  472M     0   13%  /boot
# tmpfs                               201M     0  201M    0%  /run/user/1000

```

These days the default in Ubuntu systems build with **LVM** (Logical Volume Manager) which provides an additional layer of software abstraction between the file system and the disks and by default the **/boot** partition is created pretty small. Resizing LVM volumes is an involved process at best, and whilst the **472mb** doesn't sound like a lot, it should be more than enough to store our boot data.

## What's Your Kernel and What's In /boot

In brief, the **/boot** directors contains a files relating to booting the OS, these are the the **Linux Kernel,** a temporary file system used to boot (prior to the loading of the kernel), symbol lookup maps and the bootloader (typically GRUB) within **/boot/grub**. All of these files are numbered to match the version of the kernel that they relate to.

Given the ability in Linux to switch between versions of the loaded kernel at boot, it makes sense that you may wish to keep several versions available at once. Before we start removing anything, we'll need to know what kernel we're using so we don't trash our active kernel, we can find this out with **uname**:

```bash
uname -r
# 4.4.0-87-generic
```

So we really don't want to kill that kernel, on the surface you would think you could delete anything in **/boot** that doesn't match that pattern, right? Well...that can leave you in a bit of a mess, and break your package manager, and break your bootloader. Don't do that. It's so common for people to wreck this partition that the Ubuntu docs have a small paragraph in their Kernel Management documentation named [**Oops, Removed All Kernels**](https://help.ubuntu.com/community/RemoveOldKernels#Oops.2C_Removed_All_Kernels.21).

## Safely Clearing Kernels and Resolving

We should **always** manage our installed kernels using the package manager, if the kernels aren't actually in use then using the **apt-get autoremove --purge** will remove them as part of it's action of removing all unused packages, however if you're already in this situation and you got here because your **upgrade** action tried to stuff one last kernel in the partition, this won't work because you're already at capacity and the pending install will try and run **before** the removals.

So, before we can do anything, lets see what kernels the package manager thinks we have installed, we can filter the output very nicely using some grep and tail expressions:

```bash
dpkg -l | tail -n +6 | grep -E 'linux-image-[0-9]+'
# ii	linux-image-4.4.0-157-generic	4.4.0-157.185	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-169-generic	4.4.0-169.198	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-170-generic	4.4.0-170.199	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-171-generic	4.4.0-171.200	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-174-generic	4.4.0-174.204	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-178-generic	4.4.0-178.208	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-179-generic	4.4.0-179.209	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-184-generic	4.4.0-184.214	amd64	Signed kernel image generic
# ii	linux-image-4.4.0-87-generic	4.4.0-87.110	amd64	Linux kernel image for version 4.4.0 on 64 bit x86 SMP
```

As our initial error occurred trying to install one more kernel, we can realistically get around this by:

1. Removing one (or at a push two) kernels manually
2. Completing our upgrade process (assuming one was in progress)
3. Fixing any broken packages in the package manager
4. Completing a purge on the package manager to remove any unused packages (including unused kernels)

So let's go about that, in our example we're going to remove **linux-image-4.4.0-157-generic**, ensure that the correct version pattern is used I.E. 4.4.0-157

```bash
# Remove the kerenel init file
sudo update-initramfs -d -k 4.4.0-157-generic

# Purge the kernel and it's associated "modules-extra" package
# if the package linux-modules-extra-X.X.X-XXX-generic is not specified here OR
# removed PRIOR to package linux-image-X.X.X-XXX-generic then the removal will FAIL
# due to a dependancy error
sudo dpkg --purge linux-image-4.4.0-157-generic linux-modules-extra-4.4.0-157-generic

# Fix broken packages and finish any partially completed installations
sudo apt-get -f install

# Complete the previsly started upgrade process (assuming one was in progress)
sudo apt-get upgrade

# Purge all other unused kernel packages
sudo apt-get autoremove --purge
```

That last step in particular can take some time to complete, as all except currently used kernels (among probably a lot of other packages if the system has not been very well maintained) will be purged, following this we should see the **/boot** partition look a lot healthier and be able to install additional packages are required.
