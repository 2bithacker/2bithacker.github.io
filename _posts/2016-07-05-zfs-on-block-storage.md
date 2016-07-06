---
layout: post
title: "ZFS on DigitalOcean Block Storage"
description: ""
category: hackery
tags: ['freebsd','zfs','digitalocean']
---
[DigitalOcean](https://www.digitalocean.com/) presently has a [block storage](https://www.digitalocean.com/features/storage/) feature in the beta stage. I've been doing some poking around with it using FreeBSD and ZFS, and it makes for an easy to set up and expand storage solution. Here's a quick walkthrough of how to use block storage volumes with FreeBSD for ZFS, including encryption.

For the initial setup:

* FreeBSD 10.2 droplet w/ 4GB of RAM
* 100GB block storage volume attached

Why 4GB? Mostly because ZFS becomes unstable if you don't have enough RAM. 4GB is generally a good minimum, but may still require some tuning if you're running a lot of other processes on the box.

So, what does that look like?

```
% dmesg | grep ^da
da0 at vtscsi0 bus 0 scbus2 target 0 lun 1
da0: <DO Volume 1.5.> Fixed Direct Access SPC-3 SCSI device
da0: 300.000MB/s transfers
da0: Command Queueing enabled
da0: 102400MB (209715200 512 byte sectors: 255H 63S/T 13054C)
```

Before moving ahead with the setup, let's load the [aesni](https://www.freebsd.org/cgi/man.cgi?query=aesni&sektion=4) driver to enable hardware accelerated AES encryption.

```
% sudo kldload aesni
```

And now we can proceed with setting up a partition, [geli](https://www.freebsd.org/cgi/man.cgi?query=geli&sektion=8) encryption on that partition, and a ZFS pool:

```
% sudo gpart create -s gpt da0
da0 created

% sudo gpart add -t freebsd-zfs -l volume-nyc1-01 da0
da0p1 added

% sudo geli init -l 256 /dev/gpt/volume-nyc1-01
Enter new passphrase:
Reenter new passphrase:

Metadata backup can be found in /var/backups/gpt_volume-nyc1-01.eli and
can be restored with the following command:

	# geli restore /var/backups/gpt_volume-nyc1-01.eli /dev/gpt/volume-nyc1-01

% sudo geli attach /dev/gpt/volume-nyc1-01
Enter passphrase:

% sudo zpool create tank /dev/gpt/volume-nyc1-01.eli
```

Note that the label I'm using for the block device matches the default name that DigitalOcean gives to the volume. I figured this is probably a good habit to keep things straight if I decide to move block devices around in the future.

Also, since accessing this block device is probably going to be slower than the local SSD, I figure it makes sense to enable ZFS file system compression in order to minimize the amount of data being transferred.

```
% sudo zfs set compression=lz4 tank
```

At this point, here's what the various moving pieces look like:

```
% geli list
Geom name: gpt/volume-nyc1-01.eli
State: ACTIVE
EncryptionAlgorithm: AES-XTS
KeyLength: 256
Crypto: hardware
Version: 7
UsedKey: 0
Flags: NONE
KeysAllocated: 200
KeysTotal: 200
Providers:
1. Name: gpt/volume-nyc1-01.eli
   Mediasize: 107374147584 (100G)
   Sectorsize: 512
   Mode: r1w1e1
Consumers:
1. Name: gpt/volume-nyc1-01
   Mediasize: 107374148096 (100G)
   Sectorsize: 512
   Stripesize: 0
   Stripeoffset: 17408
   Mode: r1w1e1

% zpool status
  pool: tank
 state: ONLINE
  scan: none requested
config:

    NAME                      STATE     READ WRITE CKSUM
    tank                      ONLINE       0     0     0
      gpt/volume-nyc1-01.eli  ONLINE       0     0     0

errors: No known data errors

% zpool list
NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank  99.5G  98.5K  99.5G         -     0%     0%  1.00x  ONLINE  -

% zfs list
NAME   USED  AVAIL  REFER  MOUNTPOINT
tank    61K  96.4G    19K  /tank

% df -h
Filesystem         Size    Used   Avail Capacity  Mounted on
/dev/gpt/rootfs     57G    2.2G     50G     4%    /
devfs              1.0K    1.0K      0B   100%    /dev
tank                96G     19K     96G     0%    /tank
```

Now let's say time passes, you've started storing stuff on the volume and decide you need more space. Let's add a second volume to the pool (after creating and attaching the volume through the web UI):

```
% dmesg | grep ^da1
da1 at vtscsi0 bus 0 scbus2 target 0 lun 2
da1: <DO Volume 1.5.> Fixed Direct Access SPC-3 SCSI device
da1: 300.000MB/s transfers
da1: Command Queueing enabled
da1: 102400MB (209715200 512 byte sectors: 255H 63S/T 13054C)

% sudo gpart create -s gpt da1
da1 created

% sudo gpart add -t freebsd-zfs -l volume-nyc1-02 da1
da1p1 added

% sudo geli init -l 256 /dev/gpt/volume-nyc1-02
Enter new passphrase:
Reenter new passphrase:

Metadata backup can be found in /var/backups/gpt_volume-nyc1-02.eli and
can be restored with the following command:

	# geli restore /var/backups/gpt_volume-nyc1-02.eli /dev/gpt/volume-nyc1-02

% sudo geli attach /dev/gpt/volume-nyc1-02
Enter passphrase:

% sudo zpool add tank /dev/gpt/volume-nyc1-02.eli
```

And that's it, the filesystem automatically expands to the size of the pool.

```
% zfs list
NAME   USED  AVAIL  REFER  MOUNTPOINT
tank  5.91G   187G  5.91G  /tank
```

That's it. It's kindof stupidly easy to set up and start using, and to expand as needed. Since DigitalOcean's block storage has it's own redundancies, there's no need to use mirroring or RAID at the ZFS layer.
