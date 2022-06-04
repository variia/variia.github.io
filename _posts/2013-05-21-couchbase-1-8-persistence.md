---
title: Couchbase 1.8 Persistence
date: 2013-05-21T11:34:53+00:00
author: Iván Ádám Vári 
layout: post
permalink: /couchbase-1-8-persistence/
desription: Couchbase persists every key on disk but incorrect expiry could push resident ratio down, disk usage up and re-claiming the disk isn't that simple
keywords: couchbase,disk,fragmentation,persistence,maintenance
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
<a href="http://www.couchbase.com" target="_blank">Couchbase</a> 1.8 supports two types of buckets but the _memcached_ bucket is limited, does not
support persistence, failover so this article is about the _couchbase_ bucket type and its maintenance.

We tend to forget the fact, that this bucket is persisted so every single key is saved to disk. This means you have a copy in memory (assume your
resident ratio is 100%) and on disk. Depending on your cluster setup, you will likely to have at least another copy in another node's memory
and its disk. (4 copies altogether)

With the added metadata overhead, it's fair to say that you actually need more disk space on each node than memory, to be able to fully utilise your
node's memory and you have to consider this when you size your hardware. Couchbase 2.x requires even more disk-space (2 x your RAM) per node due to
the JSON indexes and changed persistence layer.

>The behavior/technique explained here only true up to a certain size, aka vacuum is only practical for smaller databases. For large databases
(10G+ per file), it's much more efficient to fail over the node then add it back to the cluster followed by rebalance.
{: .prompt-info }

## The Keys

Depending on what you use a _key-value_ store for, it's fairly common, that the data you store changes frequently and how long it will remain in your
bucket depends on the expiry you set on creation.

_You have to nail the expiry to prevent bloating the store or deleting important dataset too early_

We started loading one of our high performance buckets and set the expiry initially to 30 days. Unfortunately we didn't realize at that point, that we
get more data from web visitors, than we delete (by expiry) so our bucket got bloated with approximately 500 - 600 Million keys. (39G in memory, 45G on
disk per node) This pushed our resident ratio down to 24% approximately, which was not really an issue until we started actually fetching data for our
testing.

## The Cleanup

We could have raised the bucket quota since we had free RAM, but we knew the bucket was bloated with stale data, so the only option we had was to clean
it up. We set the expiry immediately in our code to 7 days, but it only affected newly created keys so we had to wait 4 weeks before we actually saw
our store reducing in size.

![Bucket](/assets/img/2013-05/971B1CF6-BE65-11E2-9FAE-001D095D855C.png)

Our bucket **started reducing** in size around January **but only in memory**, the green area representing the disk usage remained the same.

By early February we reached our target size, but our disk was still bloated and it had to be claimed back, although it was not as simple as you may think.

## SQL Fragmentation

Couchbase 1.8 uses SQLite3 to persist the data, it comes with the product installer package. The <a href="http://support.couchbase.com/entries/21724787-Understanding-SQLite-fragmentation-and-Vacuuming-of-databases" target="_blank">knowledgebase</a>
(login required) from the support portal explains this very well although in real life scenario, I believe there is a little more to it.

## The not so Easy Way

Make sure you have _auto-failover_ set, remove the node to be cleaned followed by _rebalance_. When finished add the node back to the cluster and complete _rebalance_ again.
When you initiate the _rebalance_ operation the entire Couchbase data area on your disk is deleted, keys and metadata will be synced across all live nodes.

>Even during off peak production, this is a very heavy operation
{: .prompt-warning }

**Every node** will be hitting the disk hard, CPU usage will increase, response times and latency will be poor. One node took approximately 2 x 20 minutes,
on our small 4 node cluster it's 160 minutes and it's best case scenario. When fully utilized, this could go up as high as 12 hours just to clean our 4 node
cluster and it's worth to mention that we have the fastest SSD available.
The final important aspect of this method is that rebalance fails sometimes, there is a lot going on and it's somewhat _normal_ according to 
<a href="http://blog.couchbase.com/top-10-things-ops-sys-admin-must-know-about-couchbase" target="_blank">another article</a>. When it does you have to start
the _rebalance_ all over again, and it could go on for some time until you get all your nodes cleaned. This is something we could not afford, it's just not
feasible for 24/7 operations.

## Manual fragmentation

The idea is that you essentially make the data unavailable on a single node until the _VACUUM_ runs then you just suck the data back into memory from the
de-fragmented store. Yes, your active-replica will not be activated and in our case (4 node cluster) 25% of our data will not be accessible at all but for us
it is better than having uselessly slow 100% while _rebalance_ is running.

This requires _auto failover_ to be turned off, you can do it by a simple API call:

```bash
$ curl "http://localhost:8091/settings/autoFailover" -i -u Administrator:"yourpassword" -d 'enabled=false'
```

followed by shutting the _coushbase-server_ process down. When your server process is stopped, you can then _VACUUM_ the database files individually as explained
in the <a href="http://support.couchbase.com/entries/21724787-Understanding-SQLite-fragmentation-and-Vacuuming-of-databases" target="_blank">knowledgebase</a> article.

## The Surprise

It may fail with the following error:

`Error: disk I/O error`

After some reading and digging, we found that SQLite3 makes a copy of your database on the fly to `/var/tmp` then runs the VACUUM over the temporary copy, on success
it copies the de-fragmented database back to its place. Our database files were ~35-40GB each and as much as we pay attention to partitioning when we build servers,
we did not count that into our sizing. Not to mention that `/var/tmp` was not on SSD storage so the VACUUM was slow too, hence we ended up with the following fix for
our disk layout:

```
/dev/sdb1 on /opt type ext4 (rw,noatime,data=writeback,commit=120)
/opt/temp on /var/tmp type none (rw,bind)
```

Couchbase lives under `/opt` default so we mounted our fast SSD disk (sdb) there during installation, thus we just added some performance tuning options to our EXT4
file system.

Then we created a temporary folder at `/opt/temp` and mounted `/var/tmp` to it. This way we got enough space to complete the database _VACUUM_ and we moved this high
IO operation to our fastest drive available.

At completion, start the server process and <a href="http://www.couchbase.com/docs/couchbase-manual-1.8/couchbase-monitoring-startup.html" target="_blank">monitor</a>
the warmup. This method was not only faster, but leaves 75% of our data intact and fast not to mention that we do not have to risk wasting time on _rebalance_ failures.
At last but not least re-enable your _auto failover_ when all of your nodes are done:

```bash
$ curl "http://localhost:8091/settings/autoFailover" -i -u Administrator:"yourpassword" -d 'enabled=true&timeout=30'
```
