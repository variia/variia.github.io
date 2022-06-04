---
title: Solving Camel ActiveMQ Clients in TCP TIME_WAIT
date: 2014-06-04T21:25:27+00:00
author: Iván Ádám Vári
layout: post
permalink: /solving-camel-activemq-clients-in-tcp-time_wait/
description: Solving camel activemq clients in tcp time_wait for java applications that make large number of asynchronous request.
keywords: camel,client,activemq,tcp,time_wait,solved
categories:
  - Java
tags:
  - activemq
  - camel
  - java
comments: true
sharing: true
footer: true
---
We are an agile software development company and agile is great for "moving target". We plan, work and implement changes in small batches and ongoing re-factoring is just
the nature of what we do.

We recently added some functionality as well as increased traffic for one of our Java products utilising <a href="http://camel.apache.org" target="_blank">Apache Camel</a>
and <a href="http://activemq.apache.org" target="_blank">ActiveMQ</a>. The product has been in production for years now, functioning with very much zero defect rate. Not soon
after deploying the new code, our monitoring system triggered alerts about unusually high TCP TIME_WAIT connection states on the server where the new code was running. We
began the troubleshooting process and found they were all ActiveMQ connections to our broker. Our developers immediately confirmed that

"_there was no change on the ActiveMQ connection manager side._"

Well, it turned out that it was exactly the problem.

## Solving Camel ActiveMQ Clients in TCP TIME_WAIT

I started looking various aspects of our environment but was unable to pinpoint where the problem was coming from. So I shifted my focus onto our client implementation, despite
the confirmation from the developers.

![TCP Sockets by week](/assets/img/2014-06/DC3DA587-A560-4F29-97E1-B814380F14DE.png)

Note: it's perfectly natural to have these especially on systems that deal with lots of short lived requests from client connections over unreliable public networks. In nutshell,
the local TCP stack waits for twice the maximum segment lifetime (MSL) to pass (120 sec default) before it finishes CLOSING to be sure that the remote end-point received the
acknowledgement (and was not queued on upstream routers). Normally it's harmless, although in large volume could cause memory overflow.

We did have enough <a href="http://en.wikipedia.org/wiki/Ephemeral_port" target="_blank">ephemeral ports</a> to support 3.5K TIME_WAIT sockets, my issue was that it was an
extra ~1K and coming from my local network.

While looking at our code, I spotted something interesting in our client implementation, we used <a href="http://activemq.apache.org/maven/apidocs/org/apache/activemq/spring/ActiveMQConnectionFactory.html" target="_blank"><em>ActiveMQConnectionFactory</em></a>
instead of <a href="http://activemq.apache.org/maven/apidocs/org/apache/activemq/pool/PooledConnectionFactory.html" target="_blank">PooledConnectionFactory</a>. Although our
application was functioning, large volume of asynchronous messages created overhead around socket maintenance on server what we don't need. After replacing our code to use
PooledConnectionFactory, we loaded the application into our test environment to confirm the affect.

ActiveMQConnectionFactory (old):

  * 24 ESTABLISHED connections to the broker
  * ~150 TIME_WAIT sockets after the initial startup burst of ~1000

PooledConnectionFactory (new):

  * 6 ESTABLISHED connections to the broker
  * 0 TIME_WAIT sockets

We managed to reproduce this 100% in our test environment, and deploying the new code into production had not affected our throughput either.

![TCP Sockets by day](/assets/img/2014-06/AC3DC564-HDFG-4829-97E1-B814380K1421.png)

>Camel, which uses Spring JMS underneath benefits from pooling aware JMS ConnectionFactory such as PooledConnectionFactory hence it is always recommended.
{: .prompt-tip }
