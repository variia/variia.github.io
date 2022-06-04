---
title: Hands-on with Couchbase
date: 2013-05-12T07:42:09+00:00
author: Iván Ádám Vári 
layout: post
permalink: /hands-on-with-couchbase/
desription: Story of our project, migrating from Voldemort to Couchbase. Great hands-on experience, real teamwork and the benchmarking results speak for themselves.
keywords: couchbase,client,performance,speed,latency,scaling,benchmarking,scaling,clustering
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
It's been a long a long time coming, hard work has finally paid off and the last 7 months feels like just only few weeks.
<a href="http://www.couchbase.com" target="_blank">Couchbase</a> is now our primary NoSQL (key-value) store for production
and we are impressed with the results. This article is about our hands-on experience, benchmarking results and its associated
challenges.

## Background

We work in the online advertising market and for today's internet user speed is everything. Therefor, latency is paramount in
our application design thus, we needed something fast to store various user information including targeting data.
Few years ago, the choice was <a href="http://www.project-voldemort.com" target="_blank">Voldemort</a> for its latency and
speed, but unfortunately the product was not only vulnerable to cluster changes and disasters, but also was featuring small
user group so support was difficult. `memcached` always looked promising but the lack of clustering and disk persistence was
_too expensive_ for our production suite.

Then Couchbase (membase with persistance backend) came along which was pretty new on the market, was going through couple of
re-brandings in a short period but it used `memcached` under the hood along with seamless clustering, auto-recovery and disk
persistence. Sounds like a dream? Well it was, but we had to wake up quickly in the middle of our migration because it was
just not going the way we wanted to.

## The Cluster

As usual, one of our development team grabbed the Java SDK and started working on the code to implement a client in our production
suite. In the meantime we carved a new cluster out of 4 servers with the following specs:

  * HP DL 360G7 Server
  * 12 cores
  * 256GB RAM
  * 200GB SSD for disk persistence
  * CentOS 6.1
  * Couchbase Server 1.8.0

It appeared to be a capable beast and after a short period, we started populating data to it in parallel with the existing Voldemort stores.
We loaded approximately 160 Million, keys to one of our high throughput buckets within a matter of weeks.

## The Failure

Initial tests showed poor latencies of `5ms` and our application was throwing zillions of errors within seconds so we started refactoring.
We added `MBeans` to have visibility, gone over the SDK documentation number of times, added debugging metrics and it seemed whatever we do,
it is just not working out for us. We began monitoring the servers themselves and noticed that our utilization (context switching, interrupts)
do not seem to be affected by our application tests, so I started believing that we trip somewhere so early, that we don't even make it to the
"buckets".

This went on for some time, even engaged the commercial support which was helpful to pinpoint potential issues in our setup, but we have not really
got much difference in results and our team really exhausted all of its options at that point of time. Couple of months later we felt that it's time
to look for something else and my team began looking at <a href="http://www.aerospike.com" target="_blank">AeroSpike</a> with the exception of me.
I simply refused to give up and was unable to accept failure.

## The Breakthrough

Without much motivation I continued looking, and after so many hours of troubleshooting, Google searching I bumped into something promising we should
have (perhaps) looked at much earlier, the <a href="http://http://www.couchbase.com/wiki/display/couchbase/Java+Load+Generator" target="_blank">Java Load Test Generator</a>
from the community wiki. Disregarding our "sprint" plans, I grabbed a couple of developers along with the code and started a (somewhat) secret project
to work on a "real load tester" application. The code was a bit hard to read but we managed to modify it so it worked with our keys from the pre-populated
(160 million) data bucket not with some random generated garbage. We fired it from a single node and immediately managed to squeeze more juice out of the
cluster than ever before.

This is when the real development commenced with real, production data, on our production suite although we carried out the exercises off peak for our
customers's safety. We had to play with the setup to find the sweet spot and after a day of testing, we finally made a breakthrough. Of course it became
obvious that our `client` implementation was the problem not the product but without having another implementation alongside and of course enough
hands-on experience, it was impossible to prove. In a couple of weeks time, we completely refactored our client implementation based on the load tester
and I am happy to say that it's in production now for a good couple of months without a single glitch.

## Benchmark setup

  * Extracted 2 million keys out of our production bucket into a flat file, the load test client read this file into memory during startup
  * Ensured that all of our buckets are 100% memory resident (in-memory store only), even with SSDs reading is just too costly for us
  * Upgraded our cluster to version 1.8.1 and also patched it with <a href="http://support.couchbase.com/entries/21374979-TAP-disconnect-causes-memory-leak-in-1-8-x-MB-6550-" target="_blank">Hotfix MB-6550</a>
  * Vacuumed all of our persisted databases on disk (SQLite) to ensure maximum performance (our production data population, [write only]
    was constantly running during our benchmarks tests)
  * Copied the modified `ycsb.jar` along with the rest of the load test code to our NFS share
  * Adjusted the JVM to use 1G Max Heap along with the <a href="http://www.couchbase.com/docs/couchbase-sdk-java-1.0/java-gc-tuning.html" target="_blank">recommended JVM options</a> however the difference was neglectable
  * Executed the load test from 20 production application servers, all at once

Test configuration:

```
db=com.yahoo.ycsb.db.MembaseClient
memcached.address=192.168.1.1
memcached.port=11211
histogram.buckets=20
exportfile=/tmp/results.txt
recordcount=100000
operationcount=2000000
workload=com.yahoo.ycsb.workloads.MemcachedCoreWorkload
readallfields=true
insertproportion=0
readproportion=0.100
updateproportion=0
scanproportion=0
memgetproportion=0.100
memupdateproportion=0.0
valuelength=2048
workingset=100000
churndelta=100000
printstatsinterval=5
requestdistribution=zipfian
threadcount=8
```

JVM tuning:

```bash
$ java -Xmx1g -XX:+UseConcMarkSweepGC -XX:MaxGCPauseMillis=850 -cp "lib/*:build/*" com.yahoo.ycsb.LoadGenerator -t -P loadtest.cfg
```

## Results and Conclusion

  ![Console](/assets/img/2013-05/388BBE55-372F-4C56-86C8-61334B06579A.png)

  * Couchbase scaled very well during our test, we achieved **400 000 GET requests / second** with an average latency of **400us (micro-second)**, 99th = 4ms
    without using Multi-GET
  * CPU usage was not too heavy although we observed **50K to 120K interrupts / second** during load testing (per cluster node)
  * Increasing the client side **threads beyond 16** doubled the latency with **only 20% increase** in throughput, although it is likely to be caused by the
    limitation of the client application (or its hardware)

Results:

```
[OVERALL] RunTime(ms), 30008.0
[OVERALL] Throughput(ops/sec), 66648.89362836577
[GET] Operations, 2000000
[GET] AverageLatency, 401.87us
[GET] MinLatency, 146.0us
[GET] MaxLatency, 2.11s
[GET] 95thPercentileLatency, 2ms   
[GET] 99thPercentileLatency, 4ms   
[GET] 99.9thPercentileLatency, 16ms  
[GET] Return=0, 1989077
[GET] Return=-1, 10923
[GET] 0us    - 256us , 732757
[GET] 256us  - 512us , 1071121
[GET] 512us  - 1024us, 159357
[GET] 1024us - 2ms   , 22602
[GET] 2ms    - 4ms   , 2699
[GET] 4ms    - 8ms   , 10831
[GET] 8ms    - 16ms  , 403
[GET] 16ms   - 32ms  , 38
[GET] 32ms   - 64ms  , 0
[GET] 64ms   - 128ms , 160
[GET] 128ms  - 256ms , 0
[GET] 256ms  - 512ms , 0
[GET] 512ms  - 1024ms, 0
[GET] 1024ms - 2s    , 12
[GET] 2s     - 4s    , 20
[GET] 4s     - 8s    , 0
[GET] 8s     - 16s   , 0
[GET] 16s    - 32s   , 0
[GET] 32s    - 64s   , 0
[GET] 64s    - 128s  , 0
[GET] &gt;128s  , 32</pre>
```

## Final Verdict

It seemed that Couchbase really shines between **15% (~75K) and 80% (400K)** of it's max throughput (520K). Low volume buckets (1-2 GETs / second or less)
only reach 600 - 700us regardless of disk write queue, resident ratio, etc. This suggests that perhaps there is some kind of _prefetch mechanism_ for
actively requested data sets so if you request something frequently, you get better results. Adding more nodes with less memory would increase throughput,
improve bucket sharding, reduce network generated interrupts.

>Couchbase performs better under pressure but you have to keep the pressure within 80% of your maximum throughput to maintain lowest latency.
{: .prompt-tip }

