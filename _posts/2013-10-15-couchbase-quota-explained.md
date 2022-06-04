---
title: Couchbase Quota Explained
date: 2013-10-15T22:28:13+00:00
author: Iván Ádám Vári
layout: post
permalink: /couchbase-quota-explained/
description: Couchbase quota is a simple thing but misunderstanding or misconfiguring its limits can have impact on your performance or data integrity.
keywords: couchbase,quota,explained,persitence,memory limits,resident ratio
categories:
  - NoSQL
tags:
  - cluster
  - couchbase
  - nosql
comments: true
sharing: true
footer: true
---
For modern, high performance web applications we need low latency and Couchbase excels in that. To maintain the lowest possible latency even during node failure,
we need to achieve 100% resident ratio for our high performance buckets. This means that Couchbase serves all your data from RAM, even the least frequently accessed ones,
disk is used for persistence only. It turns out that in this condition your usable RAM is lot less, 2 thirds of your allocated quota.

I always found it hard to come up with a number for bucket quota. It depends on the key size which can vary, the number of keys expected to be stored, your expiry
on keys, etc. So what we do is essentially oversize the buckets initially (as long as we can afford it with RAM of course), then observe and perhaps adjust it later on
but this gets harder when you start utilising your cluster RAM.

![Quota](/assets/img/2013-10/CFDD97C6-310B-11E3-AD2F-001D095D855C.jpg)

I had a solid understanding of the "mem\_low\_wat" and "mem\_high\_wat" watermarks, I just didn't know their impact and how their values are calculated.
The <a href="http://www.couchbase.com/docs/couchbase-manual-2.0/couchbase-introduction-architecture-ejection-eviction.html" target="_blank">working set management guide</a>
explains this, I am not going to repeat what is discussed in that, just reflect what happens in real life.

In my case the default ratio (per bucket):

**mem\_low\_wat = ~60%** (of your dynamic RAM quota)

**mem\_high\_wat = ~75%** (of your dynamic RAM quota)

> In nutshell: when a certain amount of RAM is consumed by items, the server will eject items starting with replica data. This threshold is known as the low water mark.
> If higher threshold is breached, Couchbase Server will not only eject replica data, it will also eject less-frequently used items.
>
> This ejection happens at node level.

I have always monitored both, the resident ratios and of course the quotas but missed the "low" and especially the "high" watermarks. This turned out to be problematic
in the last couple of weeks, our cluster lost it's harmony and we started fetching from disk which adds an extra 100-200us latency to our average GETs occasionally.
One of my buckets is somewhat large, currently the quota is set to 100GB (400GB across 4 nodes) and the bucket holds 500+ Million keys. If you consider the ratio above,
the default behaviour sets the "mem\_high\_wat" to 100GB, that is a lot of RAM to be wasted!

**Explanation:**

In layman's term:  it's a buffer, it's a protection against unexpected large write spike, node failure, manual failover, rebalance, etc. If the bucket's memory usage
reaches 90% of its quota allocation, the application would start throwing "temporary out of memory errors" which would of course need to be handled some ways.
If an application is not coded to handle this type of exception, writes/reads may fail.

>While these watermarks can be changed on per node / per bucket basis, it's strongly recommended not to do so as it can potentially put your data at risk.
{: .prompt-info }

