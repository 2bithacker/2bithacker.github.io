---
layout: post
title: "ZFS on DigitalOcean Block Storage"
description: ""
category: hackery
tags: ['freebsd','zfs','digitalocean']
---
[DigitalOcean](https://www.digitalocean.com/) presently has a [block storage](https://www.digitalocean.com/features/storage/) feature in the beta stage. I've been doing some poking around with it using FreeBSD and ZFS, and it makes for an easy to set up and expand storage solution. Here's a quick walkthrough of how to use block storage volumes with FreeBSD for ZFS, including encryption.

### Creating the droplet

First off, we need a FreeBSD droplet. I'd recommend at least 4GB of RAM, as ZFS tends to be very memory intensive, especially if you're interested in doing block de-duplication. With less than 4GB, FreeBSD will disable ZFS prefetch by default, and under 2GB will likely need special tuning to remain stable. For more on that topic, see the [ZFS Tuning Guide](https://wiki.freebsd.org/ZFSTuningGuide) in the FreeBSD wiki.

```
> doctl compute droplet create storage-test-01-nyc1 --region nyc1 --enable-ipv6 --size 4gb --image freebsd-10-2-x64 --ssh-keys 1797332 --wait
ID		Name			Public IPv4	Memory	VCPUs	Disk	Region	Image		Status	Tags
19242119	storage-test-01-nyc1	192.0.2.91	4096	2	60	nyc1	FreeBSD 10.2	active
```

### Creating a volume

Now that we've got a FreeBSD droplet, let's create a volume to use with it. We just need to specify the volume size and region, and give it a name. This name is just a label to identify the volume and can be whatever you like.

```
> doctl compute volume create volume-nyc1-01 --size 100GiB --region nyc1
ID					Name		Size	Region	Droplet IDs
f5066553-46e4-11e6-acf9-000f53315870	volume-nyc1-01	100 GiB	nyc1
```

Now that the volume exists, we need to attach it to the droplet, so we just take the volume ID and droplet ID from their create commands and feed them into the following:

```
> doctl compute volume-action attach f5066553-46e4-11e6-acf9-000f53315870 19242119
ID		Status		Type		Started At			Completed At	Resource ID	Resource Type	Region
120277882	in-progress	attach_volume	2016-07-10 21:27:42 +0000 UTC	<nil>		0		backend		nyc1
```

We can now log into the droplet (ex. `doctl droplet ssh freebsd@storage-test-01-nyc1`) and confirm the volume has been attached by looking at `dmesg` for da0:

```
% dmesg | grep ^da0
da0 at vtscsi0 bus 0 scbus2 target 0 lun 1
da0: <DO Volume 1.5.> Fixed Direct Access SPC-3 SCSI device
da0: 300.000MB/s transfers
da0: Command Queueing enabled
da0: 102400MB (209715200 512 byte sectors: 255H 63S/T 13054C)
```

Just as a quick note, the local SSD for a FreeBSD droplet appears as 'vtbd0' and any volumes attached will appear as 'da' devices.

### Partitioning the volume

While not strictly necessary, since we'll be using the entire volume for a single filesystem, it's generally a good idea to put a partition map on the volume. This allows for meaningful labels to be applied among other things. In this case we're using the GPT format:

```
% sudo gpart create -s gpt da0
da0 created
```

Then we create a single partition for ZFS, indicated by the `-t` type flag. The `-l` option is just a label for the partition and can be whatever we like. In this case, I'm having it match the volume's name to help keep things straight:

```
% sudo gpart add -t freebsd-zfs -l volume-nyc1-01 da0
da0p1 added
```

### Setting up encryption

If you're not interested in encrypting data on the volume, you can skip this section and just leave the `.eli` off the device name in later steps.

If you do want to encrypt your data, let's start by loading the [aesni](https://www.freebsd.org/cgi/man.cgi?query=aesni) driver to enable hardware accelerated AES encryption:

```
% sudo kldload aesni
```

Now we can configure [geli](https://www.freebsd.org/cgi/man.cgi?query=geli) encryption on the partition. The `-l` option here is the key length, which has to be either 128 (default) or 256 for the default AES-XTS algorithm. The passphrase entered here will be needed when the droplet is rebooted to re-attach the encrypted partition. Also note that we're referring to the partition by the label supplied earlier.

```
% sudo geli init -l 256 /dev/gpt/volume-nyc1-01
Enter new passphrase:
Reenter new passphrase:

Metadata backup can be found in /var/backups/gpt_volume-nyc1-01.eli and
can be restored with the following command:

	# geli restore /var/backups/gpt_volume-nyc1-01.eli /dev/gpt/volume-nyc1-01
```

Now that geli has been initialized, we need to attach the encrypted partition to the system. This step will need to be done when the system is rebooted. The passphrase entered here has to match the phrase used during initialization.

```
% sudo geli attach /dev/gpt/volume-nyc1-01
Enter passphrase:
```

This will set up `/dev/gpt/volume-nyc1-01.eli`, which is the decrypted version of the partition. Data written to that block device is encrypted and written out to the underlying device.

### Setting up ZFS

Now for the easy part, creating the ZFS pool! Since DigitalOcean volumes implement their own data redundancy, there's no need to create multiple volumes and mirror them, or to run them in a RAID-Z configuration, we can just use the individual volume directly in the pool. We're using the generic name of 'tank' for our pool, but again, it can be pretty much anything you like.

```
% sudo zpool create tank /dev/gpt/volume-nyc1-01.eli
```

Since the volume is attached over a network, access to it is going to be slower than the local SSD. In order to minimize the amount of data being written to the device, we can enable compression at the ZFS filesystem layer. This is entirely optional, and can be set on a per-filesystem basis. In this case, we're using the 'lz4' compression algorithm, which is optimized for speed while still giving decent compression. For other options, consult the [zfs](https://www.freebsd.org/cgi/man.cgi?query=zfs) man page.

```
% sudo zfs set compression=lz4 tank
```

At this point, we can now look and see the ZFS pool with it's total size being slightly smaller than the total volume size, due to partitioning and formatting overhead:

```
% zpool list
NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank  99.5G  98.5K  99.5G         -     0%     0%  1.00x  ONLINE  -
```

And the ZFS filesystem in that pool, both through the ZFS list command and a regular `df`:

```
% zfs list
NAME   USED  AVAIL  REFER  MOUNTPOINT
tank    61K  96.4G    19K  /tank

% df -h
Filesystem         Size    Used   Avail Capacity  Mounted on
/dev/gpt/rootfs     57G    2.2G     50G     4%    /
devfs              1.0K    1.0K      0B   100%    /dev
tank                96G     19K     96G     0%    /tank
```

### Adding additional volumes

One of the things that ZFS makes easier is expanding the pool with additional volumes as more space as needed.

The same effect could be accomplished with UFS, but requires setting up [gconcat](https://www.freebsd.org/cgi/man.cgi?query=gconcat) and using [growfs](https://www.freebsd.org/cgi/man.cgi?query=growfs) to extend the partition, which can be tricky on a live system.

In ZFS, it's just a matter of adding an additional device to the pool.

As above, we create the new volume and attach it to the droplet:

```
> doctl compute volume create volume-nyc1-02 --size 100GiB --region nyc1
ID					Name		Size	Region	Droplet IDs
212ae50c-46eb-11e6-87c3-000f53315860	volume-nyc1-02	100 GiB	nyc1

> doctl compute volume-action attach 212ae50c-46eb-11e6-87c3-000f53315860 19242119
ID		Status		Type		Started At			Completed At	Resource ID	Resource Type	Region
120287255	in-progress	attach_volume	2016-07-10 22:10:49 +0000 UTC	<nil>		0		backend		nyc1
```

This new volume will be `da1` on the droplet. Which we then partition and label:

```
% sudo gpart create -s gpt da1
da1 created

% sudo gpart add -t freebsd-zfs -l volume-nyc1-02 da1
da1p1 added
```

And again optionally configure encryption:

```
% sudo geli init -l 256 /dev/gpt/volume-nyc1-02
Enter new passphrase:
Reenter new passphrase:

Metadata backup can be found in /var/backups/gpt_volume-nyc1-02.eli and
can be restored with the following command:

	# geli restore /var/backups/gpt_volume-nyc1-02.eli /dev/gpt/volume-nyc1-02

% sudo geli attach /dev/gpt/volume-nyc1-02
Enter passphrase:
```

And finally add it to the ZFS pool:

```
% sudo zpool add tank /dev/gpt/volume-nyc1-02.eli
```

The filesystem automatically expands to the size of the pool, which we can see in the pool and filesystem lists:

```
% zpool list
NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank   199G   229K   199G         -     0%     0%  1.00x  ONLINE  -

% zfs list
NAME   USED  AVAIL  REFER  MOUNTPOINT
tank  59.5K   193G    19K  /tank
```

And that's it. As more space is needed, more volumes can just be added into the pool.

For more information on ZFS on FreeBSD, consult the [FreeBSD Handbook](https://www.freebsd.org/doc/handbook/zfs.html).
