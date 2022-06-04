---
title: Conditional SNAT with iRule on F5
date: 2015-09-14T21:03:35+00:00
author: Iván Ádám Vári
layout: post
permalink: /conditional-snat-with-irule-on-f5/
description: Quick and dirty guide about how to create conditional SNAT with iRule on F5 and rewrite (NAT) IP addresses based on specific conditions.
keywords: f5,irule,conditional,snat
categories:
  - Network
tags:
  - f5
  - irule
  - network
comments: true
sharing: true
footer: true
---
Quick and dirty guide about how to create conditional SNAT with iRule on F5 and rewrite (NAT) IP addresses based on specific conditions.

We have 2 public IP netblocks for our production network, one is geographically registered in LA, California, the other is Amsterdam, Netherlands. It is very common that
services such as Google, Amazon, Akamai, etc serve requests based on their source but occasionally they get it wrong so I needed a way to control what netblock my request
is addressed out of.

Furthermore, some of our services require one-to-one IP mappings so I had to come up with a solution that solves the following:

  * check if the destination address is on the target list
  * check if the source of the request has one-to-one mapping for outgoing IP
  * if it does and the destination is on our target list then rewrite the address to the matched map
  * if it does not have one-to-one map and the the destination is on our target list then rewrite the address to the _default_ NAT address
  * otherwise do nothing, send packet out with its original source address

```tcl

ltm data-group internal snat_for_destination {
    records {
        8.8.8.8/32 {
            data googledns
        }
    }
    type ip
}
```

What matters here is the key (IP), the value (googledns) is just a comment although you can use it in logging statement if you want.

```tcl
ltm data-group internal snat_dmz_to_wan_map {
    records {
        1.2.3.4/32 {
            data 4.3.2.1
        }
    }
    type ip
}
```

```tcl

ltm rule snat_dmz_to_wan_map {
    when CLIENT_ACCEPTED {
    # https://devcentral.f5.com/wiki/iRules.class.ashx?lc=1
    # check IF the request's destination IP is in given match list
    if { [class match [IP::local_addr] equals snat_for_destination]} {
        # pick IP based 1-to-1 SNAT mapping for connecting client from given list
        if { [class match [IP::client_addr] equals snat_dmz_to_wan_map]} {
            # set variable for matched address object
            set snat_addr [class match -value [IP::client_addr] equals snat_dmz_to_wan_map]
            log local0. "Connection from [IP::client_addr] to [IP::local_addr] rewrite as $snat_addr \[iSNAT\]"
            snat $snat_addr
        } else {
            # on failed map lookup, default to F5's floating address
            log local0. "Connection from [IP::client_addr] to [IP::local_addr] automap as 5.6.7.8 \[iSNAT\]"
            snat automap
        }
    } else {
        forward
    }
}
```

Now, the only thing we need to do is to add this resource to the required Virtual Server object.
