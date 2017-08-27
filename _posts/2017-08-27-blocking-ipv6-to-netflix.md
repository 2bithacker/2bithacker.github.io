---
layout: post
title: "Blocking IPv6 to Netflix"
description: ""
category: hackery
tags: ['freebsd','dns']
---
Since moving out of Comcast territory and switching over to Charter Spectrum for my home Internet access, I've been forced to go back to the dark ages of getting IPv6 via HE's Tunnelbroker, which normally works great (except when people block path MTU discovery, but that's a different problem.) However, Netflix has gotten a little overzealous in blocked people using VPNs to bypass region restrictions on their content, which means anyone using Netflix via IPv6 over Tunnelbroker has been getting blocked.

Until recently I had been using a static drop rule in pf for Netflix's IP ranges, but a friend of mine pointed out dnsmasq has a nice feature to add IPs from A/AAAA requests to a pf table, so now my blocking is automated with two simple config tweaks.

In `dnsmasq.conf`:

```
ipset=/nflxvideo.net/netflix.com/nflximg.net/nflxext.com/netflix
```

In `pf.conf`:

```
table <netflix> persist
block return in quick on $trust inet6 from any to <netflix>
```

The `netflix` table will accumulate v4 and v6 addresses, but the block rule is limited to `inet6` so the IPv4 traffic will work as normal. Doing `block return` to return a `RST` to the client, so v6 will fail faster than if the initial packet just got dropped.

Also added `pfctl -t netflix -T expire 600` to cron to clear out old entries now and again.

So far this has been working well for me, hope someone else finds it useful.
