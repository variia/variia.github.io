---
title: Load Balancing and Sticky Sessions by URL Parameter
date: 2014-07-29T16:45:02+00:00
author: Iván Ádám Vári
layout: post
permalink: /load-balancing-and-sticky-sessions-by-url-parameter/
description: Choose your poison, apache or nginx. Cost effective way of reverse proxying, load balancing and sticky sessions by URL parameters for testing.
keywords: apache,nginx,reverse proxy,load balance
categories:
  - Network
tags:
  - apache
  - nginx
  - proxy
comments: true
sharing: true
footer: true
---
To be able to mimic our production workload in testing, we had to come with a low cost solution to load balance HTTP traffic between few application servers. In addition to
that, for the first (initial request) we required even distribution amongst the backend nodes but, subsequent requests needed to be handled by the same backend server.

This task was relatively easy with <a href="http://nginx.org" target="_blank">NGINX</a>, our preferred HTTP server however lately, I had to come up with a solution for
<a href="http://www.apache.org" target="_blank">apache</a> 2.2 which was not as straight forward.

## Load Balancing and Sticky Sessions by URL Parameter

apache maintains one of the best online documentation for most of its versions so finding a solution to my problem was not hard at all. However,
<a href="http://httpd.apache.org/docs/2.2/mod/mod_proxy_balancer.html#example" target="_blank">most solutions are based on cookies</a> what I could not use. The issue with
cookies is that I have limited control over them, there is no guarantee that the user has cookies enabled, or it will be saved not to mention that cookies can become corrupted too.

The other issue with cookie based load balancing is that once you have one set, you always ended up on the same backend server even when you make your _first request_.
Our requirement was that for initial calls, we want randomly allocated backend then for subsequent calls we want the user to be handled by the same server what handled the
first call.

Sticky sessions with URL parameter is supported by apache's mod_proxy:

_The second way of implementing stickyness is URL encoding. The web server searches for a query parameter in the URL of the request. The name of the parameter is specified
again using stickysession. The value of the parameter is used to lookup a member worker with route equal to that value._

For some reason, I had mixed results, over all it seemed unreliable hence I had to come up with my own solution based on mod_rewrite:

```apache
RewriteLog logs/virtualhost_rewrite.log
RewriteLogLevel 2
RewriteEngine On
RewriteCond %{QUERY_STRING} (myparam=myvalue1)
RewriteRule (.*) http://lb-node1.domain.com:8080%{REQUEST_URI} [P,L]
RewriteCond %{QUERY_STRING} (myparam=myvalue2)
RewriteRule (.*) http://lb-node2.domain.com:8080%{REQUEST_URI} [P,L]

ProxyErrorOverride On

<Proxy balancer://loadbalancer>
    BalancerMember http://lb-node1.domain.com:8080
    BalancerMember http://lb-node2.domain.com:8080
</Proxy>

ProxyPass / balancer://loadbalancer/
ProxyPassReverse / http://lb-node1.domain.com:8080/
ProxyPassReverse / http://lb-node2.domain.com:8080/
```

>Important to point out, that mod_rewrite rules are evaluated first before any other settings or rules and this behaviour is critical for this functionality.
{: .prompt-info }

We basically check the incoming URL and if we a have match on our URL parameter, we pass the request handling to mod_proxy without evaluating any other rules. Our URL parameters
are always unique and set by the backend servers so we need as much rewrite rules as backed servers we have.

The `<Proxy>` directive sets up a standard load balancer, this ensures that initial request can be load balanced evenly between all member servers. The `ProxyPassReverse`
directives ensure that we correctly pass redirects back to the clients...

The only issue with this setup is that, if one of the application servers become unreachable, the clients will receive 502 (bad gateway) if they have URL parameter set.

NGINX is much easier to set up, and since it monitors the backend servers, it can proxy traffic to other available backend servers regardless of the URL parameter being set or not.

```nginx
upstream lb-pool {
    server lb-node1.domain.com:8080;
    server lb-node2.domain.com:8080;
}

upstream node1 {
    server lb-node1.domain.com:8080;
    server lb-node2.domain.com:8080 backup;
}

upstream node2 {
    server lb-node1.domain.com:8080 backup;
    server lb-node2.domain.com:8080;
}

server {
    listen 80;
    location / {
        proxy_cache off;
        proxy_pass_header off;
        proxy_set_header Host $http_host;

        if ($arg_myparam = "myvalue1") {
            proxy_pass http://node1;
            break;
        }

        if ($arg_myparam = "myvalue2") {
            proxy_pass http://node2;
            break;
        }

        proxy_pass http://lb-pool;
}
```
