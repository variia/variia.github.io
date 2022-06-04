---
title: "macOS Optimised Battery Charging"
date: 2022-04-25 21:33:18 +0100
author: Iván Ádám Vári
layout: post
permalink: /macOS-battery-health-and-optimised-charging/
description: Manage macOS battery charging for prolonged cell life
keywords: macos,clamshell,battery,health,optimise,charging
categories:
  - macOS
tags:
  - macos
  - hardware
comments: true
sharing: true
footer: true
---
I use my Macbook in clamshell mode most of the time. Back in the day with my old Retina Macbook, I disconnected my
charger couple of times a week to discharge the battery regularly. This simple practice enabled me to use my Macbook
with an external display while conditioning the battery for maximum cell life. My battery started failing around 680 cycles (health 86%) and eventually, it was time to replace it at 873 (health 70%).

![Battery](/assets/img/2022-04/797BF636-5DDE-4852-BB63-103D6651A8FB.png){: width="250" height="270" }

As for my newer Macbook Touch, it is a different story. I have a Belkin docking station nowadays and I really liked
the type-C thunderbolt option that feeds all my power, data, display, etc to my device with a single piece of cable.
However, I am no longer be able to discharge my device regularly while in clamshell mode.

## Background

Since the arrival of the macOS release BigSur, <a href="https://support.apple.com/en-md/HT212049" target="_blank">
optimised charging</a> is built in to help your battery's lifespan, but it was too good to be true according to my
experience.

I noticed during the day, that my Macbook charged up to 80% then paused. The remaining 20% was finished later on the
afternoon when the device was less utilised. But it required the device to be discharged first, which happens mostly
on weekends for me. My battery health declined fast, which means I was losing capacity. For about 110 cycles, my
capacity was around 92% compared the original design and it declined with a higher rate than my old Retina one.

Luckily, that dodgy built-in keyboard failed (twice) on me, which resulted getting a brand new bottom case replacement
(battery, keyboard, etc) as part of this <a href="https://support.apple.com/keyboard-service-program-for-mac-notebooks" target="_blank">repair program</a>.

To avoid getting my battery worn out fast again, I started searching for solutions and I think I found something
promising.

## Solution

I bumped into this <a href="https://github.com/zackelia/bclm" target="_blank">Github project</a> which enables pausing
the charging at any level you like. To be fair, it does not solve the problem described earlier, I am still unable to
discharge my device while connected to the docking station. But at least, it allows the user to control the battery
charging to prevent overcharging while in clamshell mode. The bonus is that it is a simple, command line program available
via <a href="https://brew.sh" target="_blank">brew</a>, which can be scripted for further adjustments and management.

```bash
$ brew tap zackelia/formulae

$ brew install bclm

$ sudo bclm write 77

$ sudo bclm read 
77

$ sudo bclm 
OVERVIEW: Battery Charge Level Max (BCLM) Utility.

USAGE: bclm <subcommand>

OPTIONS:
  --version               Show the version.
  -h, --help              Show help information.

SUBCOMMANDS:
  read                    Reads the BCLM value.
  write                   Writes a BCLM value.
  persist                 Persists bclm on reboot.
  unpersist               Unpersists bclm on reboot.

  See 'bclm help <subcommand>' for detailed help.
```

As mentioned on the project's page, sources <a href="https://www.apple.com/batteries/why-lithium-ion/"  target="_blank">[1]</a>
<a href="https://www.eeworldonline.com/why-you-should-stop-fully-charging-your-smartphone-now/"  target="_blank">[2]</a>
<a href="https://www.csmonitor.com/Technology/Tech/2014/0103/40-80-rule-New-tip-for-extending-battery-life"  target="_blank">[3]</a>
claim that the Li-Ion batteries are best between 40% and 80% charge levels so I limited my device to the recommended 77%.
I have been running like this for some time and it seems, that the battery is performing well, capacity is stable.

>I cannot claim (yet), that it will prolong my battery's life but it is worth to try.
{: .prompt-info }
