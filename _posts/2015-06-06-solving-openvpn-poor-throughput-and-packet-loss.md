---
title: Solving OpenVPN poor throughput and packet loss
date: 2015-06-06T12:27:29+00:00
author: Iván Ádám Vári
layout: post
permalink: /solving-openvpn-poor-throughput-and-packet-loss/
description: This guide is about solving OpenVPN poor throughput and packet loss between two data centers and based on Couchbase XDCR experience.
keywords: openvpn,slow,throughput,solved
categories:
  - Network
tags:
  - network
  - openvpn
  - security
comments: true
sharing: true
footer: true
---
This not about optimising OpenVPN, it is about solving OpenVPN poor throughput and packet loss issue, where the server receives traffic faster than it actually process.

We are currently in the process of moving data centers. This requires our Couchbase data to be in sync between Gütersloh (DE) and AMS-IX (NL) which does mean that XDCR needs
to pump few hundred Gigs across every day and fast. After about 20 minutes or so, everything started to slow down for an unknown reason.

Our choice for VPN solution was OpenVPN due to some limitations caused by the managed network at the german side, so we built the tunnel and managed to get a reasonable link
with ~20ms TTL, initial throughput tests showed:

```
[ 4] 0.0-30.1 sec 170 MBytes 47.5 Mbits/sec
```

It was not so flash but I did not suspect anything at that point, I accepted it as the capability of the tunnel. Further investigation however revealed TX packet drops on the
tunnel interface:

```
tun1 Link encap:UNSPEC HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
 inet addr:10.0.0.129 P-t-P:10.0.0.130 Mask:255.255.255.255
 UP POINTOPOINT RUNNING NOARP MULTICAST MTU:1500 Metric:1
 RX packets:7409196373 errors:0 dropped:0 overruns:0 frame:0
 TX packets:6084526548 errors:0 dropped:2642021 overruns:0 carrier:0
 collisions:0 txqueuelen:100
 RX bytes:941878733670 (877.1 GiB) TX bytes:3395805058712 (3.0 TiB)
```

It seemed that the tunnel is not being able to keep up with the amount of traffic it received. After some reading, it turned out, that OpenVPN sets
<a href="http://en.wikipedia.org/wiki/Ifconfig" target="_blank">txqueuelen</a> parameter to 100 as default for the tunnel interfaces on both, client and server. It is essentially
a buffer, and managed by the network scheduler.

The solution was to set this to 1000, identical to the physical interface configurations:

```bash
$ grep txqueuelen /etc/openvpn/server-udp.conf
txqueuelen 1000
```

After restarting OpenVPN on both, server and client side, there was no packet drop on the tunnel interfaces and the throughput was better too:

```
[  4]  0.0-30.3 sec   267 MBytes  73.9 Mbits/sec
```

For further optimisation, visit the official <a href="https://community.openvpn.net/openvpn/wiki/Gigabit_Networks_Linux" target="_blank">Linux guide</a>.

