---
title: Merging Pillars in SaltStack
date: 2014-08-16T21:26:41+00:00
author: Iván Ádám Vári
layout: post
permalink: /merging-pillars-in-saltstack/
description: Merging Pillars in SaltStack is supported but somewhat limited. In this post, I reveal a technique to utilise Jinja2 templating for pillar manipulation.
keywords: merge,salt,pillar,saltstack
categories:
  - Python
tags:
  - python
  - salt
comments: true
sharing: true
footer: true
---
Merging or joining <a href="http://salt.readthedocs.org/en/latest/topics/pillar/" target="_blank">Pillars</a> in SaltStack is supported but somewhat limited. It took me some
time to work out a clean solution to support a specific manipulation so to make this easier, I am going to share my real life example.

## Merging Pillars in SaltStack

I wrote a reasonably complex formula to manage our DNS (bind9) servers including zone files. As a common approach, I decided to use Pillar for configuration to make the formula
generic and reusable.

My formula required the following Pillar data (YAML):

```yaml
bind:
  config:
    user: root
    group: named
    mode: 640
    custom:
      allow-query:
        - 127.0.0.1
        - 192.168.1.0/24
      allow-transfer:
        - 127.0.0.1
        - 192.168.1.0/24
      recursion: "yes"
      zone-statistics: "yes"
      transfer-format: many-answers
      interface-interval: 0
  zones:
    domain.com:
      config:
        type: master
        file: domain.com.hosts
        also-notify:
          - 192.168.1.1
          - 192.168.1.2
      soa:
        ttl: 43200 ; 12 hours
        email: hostmaster.domain.com.
        refresh: 10800 ; 3h refresh
        retry: 3600 ; 1h retry
        expire: 604800 ; 1w expire
        minimum: 10800 ; 3h minimum
        ns:
          - ns1.domain.com.
          - ns6.otherdomain.net.
          - ns7.other.sub.domain.org.
    sub.domain.org:
      config:
        type: slave
        file: slaves/sub.domain.org.hosts
        masters:
          - 192.168.1.1
          - 192.168.1.2
    sub.domain.net:
      config:
        type: forward
        forward: only
        forwarders:
          - 172.16.1.1
```

To reuse my formula, I needed slightly different pillar for each DNS server but I wanted to reuse existing details to avoid duplication and pollution hence I ended up splitting
the pillar into few files:

```
/srv/salt/pillar/base/bind/named.sls
/srv/salt/pillar/base/bind/zones-master.sls
/srv/salt/pillar/base/bind/zones-slave.sls
/srv/salt/pillar/base/bind/zones-other.sls
```

This allows the use or import of the `named.sls` (core config) for every server and depending on "role" even additional zones as required:

`/srv/salt/pillar/base/bind/named.sls`

```yaml 
bind:
  config:
    custom:
      ...
      ...
```

`/srv/salt/pillar/base/bind/zones-master.sls`

```yaml
bind:
  zones:
    domain.com:
      ...
      ...
```

First, I tried adding `init.sls` with the "include" statement:

{% raw %}
```yaml
include:
  - bind.named
{%- if grains['dnsrole'] == 'master' %}
  - bind.zones-master
  - bind.zones-other
{%- endif %}
{% endraw %}
```

This did not work as expected, actually it overrides either the first subkey of the `named.sls` (custom) or the zone file (zone) depending on which gets read first during compile.

I could have used the <a href="http://salt.readthedocs.org/en/latest/topics/pillar/" target="_blank">include statement with the nesting </a>but it only works if I restructure
the zone files and remove nesting from the bind key as well as include one zone.

One option was to move this logic into the `top.sls`:

```yaml
base:
  'roles:dns':
    - match: grain
    - bind.named
    - bind.zones-master
```

This worked perfectly, my pillars were merged exactly the I way I wanted. However, I did not want to move a messy if-else logic there to target master/slave servers differently.
Adding the if-else logic to the zone files looked not so ideal, changing the grain structure for better targeting seemed also just "too much" for what I needed to accomplish.

#### Template Engine to the Rescue

My solution was hiding <a href="http://salt.readthedocs.org/en/latest/ref/renderers/all/salt.renderers.jinja.html" target="_blank">here</a>, I just had to compose the recipe
myself. This method can be used in state files as well giving you the much needed power of code reuse.

`/srv/salt/pillar/base/bind/init.sls`

{% raw %}
```yaml
{%- import_yaml 'bind/named.sls' as named with context -%}
{%- import_yaml 'bind/zones-master.sls' as masters with context -%}
{%- import_yaml 'bind/zones-slave.sls' as slaves with context -%}
{%- import_yaml 'bind/zones-other.sls' as others with context -%}

{%- if grains['dnsrole'] == 'master' %}
{%- do named.bind.update(masters.bind) %}
{%- elif grains['dnsrole'] == 'slave' %}
{%- do named.bind.update(slaves.bind) %}
{%- endif %}

{%- do named.bind.zones.update(others.bind.zones) %}

{{ named }}
{% endraw %}
```

This imports all YAML files, compiles them as dictionaries and basically makes them available as objects by the given name. Then we just use the power of Jijna2 and carry out
a dictionary update on specific objects from a specific nested key. To me it is readable, centralised and keeps the control inside the bind pillar, yet giving the power of flexibility.

Minion targeting is now easy, keeping the `top.sls` clean and readable as possible:

```yaml
base:
  'roles:dns':
    - match: grain
    - bind
```

