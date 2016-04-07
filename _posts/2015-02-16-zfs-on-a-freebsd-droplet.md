---
layout: post
title: "ZFS on a FreeBSD Droplet"
description: ""
category: hackery
tags: ['freebsd','zfs','digitalocean']
---
Recently my employer, DigitalOcean, announced [FreeBSD droplets](https://www.digitalocean.com/company/blog/presenting-freebsd-how-we-made-it-happen/). This made me pretty happy, as I've been a long time fan of FreeBSD, running it on my home storage server, router, and a few other VPS providers. One of the things I would like to do though is make backups from my droplet to my home storage server as easy as possible. For me, this meant ZFS snapshots and being able to send snapshot diffs between systems.

Since the droplets default to UFS, the first hurdle was reinstalling the system onto a ZFS root, which turns out to be relatively simple.

Here's how I did it. This all assumes a fresh blank FreeBSD droplet, and will delete everything on the droplet in the process.

*Update 6-Oct-2015:* Finally updated this for FreeBSD 10.2!

*Update 7-Apr-2016:* Updated for FreeBSD 10.3-RELEASE.

### Create mfsBSD tarball

The key to all this is being able to boot into a memory filesystem in order to reformat the droplet's drive, and the easiest way to do that is to use [mfsBSD](http://mfsbsd.vx.sk/).

1. `fetch https://github.com/mmatuska/mfsbsd/archive/2.2.tar.gz`
2. `tar xzf 2.2.tar.gz`
3. `mkdir dist`
4. `cd dist`
5. `fetch ftp://ftp.freebsd.org/pub/FreeBSD/releases/amd64/10.3-RELEASE/base.txz`
5. `fetch ftp://ftp.freebsd.org/pub/FreeBSD/releases/amd64/10.3-RELEASE/kernel.txz`
6. `cd ../mfsbsd-2.2`
7. `make tar BASE='../dist' PKG_STATIC='/usr/local/sbin/pkg-static'` (note, this needs root)

This should leave you with a `mfsbsd-10.3-RELEASE-amd64.tar`

### Extract tarball over / and reboot

1. `cd /`
2. `tar xf /path/to/mfsbsd-10.3-RELEASE-amd64.tar`
3. (optional) edit `/boot/loader.conf` to set initial IP addrs and such
3. `reboot`

After this point, you'll want to be on the droplet's console. It should reboot into the mfsBSD environment. If you didn't edit `loader.conf` then you'll need to manually configure vtnet0 for Internet access.

### Install

Log in to the droplet's console as root, and set up networking (if you didn't change `loader.conf`)

1. `ifconfig vtnet0 inet [addr] netmask [mask]`
2. `route add default [gateway]`
3. `echo 'nameserver 8.8.8.8 >> /etc/resolv.conf`

At this point you should be able to just run `bsdinstall` and do a regular FreeBSD installation onto vtblk0, selecting ZFS when it comes to paritioning the disk.

When the install is done, just reboot again and you'll have a nice clean ZFS droplet!
