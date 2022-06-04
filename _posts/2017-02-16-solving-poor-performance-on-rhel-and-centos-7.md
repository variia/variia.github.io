---
title: "Solving Poor Network Performance on RHEL and CentOS 7"
date: 2017-02-16 17:15:57 +0100
author: Iván Ádám Vári
layout: post
permalink: /solving-poor-performance-on-rhel-and-centos-7/
description: This guide is about solving poor performance observed on RHEL/CentOS 7 running a real-time Java application.
keywords: centos,redhat,rhel,7,slow,poor,network,performance,solved
categories:
  - Network
tags:
  - linux
  - network
  - java
comments: true
sharing: true
footer: true
---
We are building the next generation online marketplace and part of it is a real-time Java application. This application is heavily optimised for its use case,
handles zillions of short-lived tcp requests fast. Most of our operations complete quickly (<100ms) and some even more quickly (<500us).

Our old application pool is based on CentOS 6 nodes and we are doing considerably well on them. However recently, we deployed our new CentOS 7 based server
farm and for some reason, we have been unable to meet the expectations set by the old pool.

We set these nodes identically, turned off CPU scaling, disabled THP, set IO elevators, etc. Everything looked fine, but the first impressions were that these new
nodes have slightly higher load and deliver less.

This was a bit frustrating.

We got newer kernel, newer OS, newer drivers, everything is set the same way, but for some reason, we just could not squeeze the same amount of juice out of these
new servers compared to the old ones.

During our observation, we noticed that the interrupts on these new nodes were much higher (~30k+), compared to the old servers but I failed to follow upon it at
that point.

## Tuned performance profiles

Up until recently, it did not occur to me that there are key differences in CentOS 7's and CentOS 6's `latency-performance` profile.

Looking at the `tuned` documentation and the profiles, it looked like there is a more advanced profile available called `network-latency`, and it is a child profile
of my current `latency-performance` profile.

```bash
$ rpm -ql tuned | grep tuned.conf
/etc/dbus-1/system.d/com.redhat.tuned.conf
/etc/modprobe.d/tuned.conf
/usr/lib/tmpfiles.d/tuned.conf
/usr/lib/tuned/balanced/tuned.conf
/usr/lib/tuned/desktop/tuned.conf
/usr/lib/tuned/latency-performance/tuned.conf
/usr/lib/tuned/network-latency/tuned.conf
/usr/lib/tuned/network-throughput/tuned.conf
/usr/lib/tuned/powersave/tuned.conf
/usr/lib/tuned/throughput-performance/tuned.conf
/usr/lib/tuned/virtual-guest/tuned.conf
/usr/lib/tuned/virtual-host/tuned.conf
/usr/share/man/man5/tuned.conf.5.gz

$ grep . /usr/lib/tuned/network-latency/tuned.conf | grep -v \#
[main]
summary=Optimize for deterministic performance at the cost of increased power consumption, focused on low latency network performance
include=latency-performance
[vm]
transparent_hugepages=never
[sysctl]
net.core.busy_read=50
net.core.busy_poll=50
net.ipv4.tcp_fastopen=3
kernel.numa_balancing=0
```

What it does additionally, is that it turns `numa_balancing` off, disables THP and makes some `net.core` kernel tweaks.

After activating this profile, we can see the effect immediately.

```bash
$ tuned-adm profile network-latency
$ tuned-adm active
Current active profile: network-latency
```

![Interrupts](/assets/img/2017-01/D97E625A-920D-40B8-9E77-2BFB1F720499.png)

My interrupts drop from a whopping 70k down to 40k, which is exactly what my CentOS 6 interrupts measure during peak load.

I could not stop there and I started experimenting with these settings and found, that the interrupts are directly affected by the `numa_balancing` kernel setting.
Restoring `numa_balancing` puts the interrupt pressure back on the system immediately.

`ENABLE numa_balancing`

```bash
$ echo 1 > /proc/sys/kernel/numa_balancing
```

`DISABLE numa_balancing`

```bash 
$ echo 0 > /proc/sys/kernel/numa_balancing
```

>Please note, that this profile is for a specific workload.
{: .prompt-info }

Turning `numa_balancing` off may improve your interrupts but it can degrade your application some other way.
Experiement with the available profiles and stick to the one that gives you the best results.
