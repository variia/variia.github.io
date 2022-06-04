---
title: Dashlane vs 1Password
date: 2014-07-16T22:33:29+00:00
author: Iván Ádám Vári
layout: post
permalink: /dashlane-vs-1password/
description: Dashlane vs 1Password, a quick and dirty comparison of the two, most secure, most well known password managers in 2014.
keywords: dashlane,1password,comparison,versus
categories:
  - Security
tags:
  - password
  - security
comments: true
sharing: true
footer: true
---
I am a sysop / devops engineer, love open source and security so I tend to ignore commercial software. For password valet, I have been using <a href="http://keepass.info" target="_blank">KeePass </a>
for years and happy with it except a couple of things:

* written in .NET so cross platform integration has its challenges
* browser integration

Although the browser integration is reasonably good now on Windows, it's not as _refined_ as its commercial competitors such as <a href="https://www.dashlane.com" target="_blank">Dashlane </a> or
<a href="https://agilebits.com/onepassword" target="_blank">1Password</a>. So I decided to investigate these utilities to see if they can convince me to switch.

## Update - July 28th, 2014:

I have to admit, that most of my concerns around 1Password were related to the earlier Windows version. Since my evaluation,
<a href="http://blog.agilebits.com/2014/06/17/1password-4-for-windows-is-here/" target="_blank">1Password 4 was made available for Windows</a> and looks, feels, behaves exactly
like on OSX.

I am very happy to say that, I made the switch and I am a satisfied 1Password customer ever since. In the end, I decided not to look at cost when it comes to security,
and <a href="http://blog.agilebits.com/2014/07/16/1password-is-a-very-safe-basket/" target="_blank">it seems that I made the right decision.</a>

## Dashlane vs 1Password

I didn't want to go into detail and compare the core or advanced features of these apps, you can find elsewhere. My goal was to find something more convenient and easier to
use than KeePass but keeping the balance between security, functionality and convenience.

It was also important to me how _trusted_ the vendor is, what kind of reputation they may have and how easy it is to get my data, should they disappear from the market.

This is purely my gut feeling / opinion and based on my quick and dirty trial based on versions available at this time of writing. I looked at how they compare to my current
software KeePass regarding functionality, security, design, look / feel and above all, how easy the transition would be.

#### Dashlane pros:

  * offline or online: you only have to be online while register an account, it does not affect working with an already created local database if you choose not to pay or
    turn off cloud sync
  * secure: 2 factor authentication supported
  * clean interface: identical look / feel on both Windows and OSX
  * security and authorisation: even if you just want to create a brand new local valet (without cloud sync) on a "new" device, it needs to be "approved" with an online
    code sent to the registered email. Dashlane essentially tracks the devices associated with an account regardless you sync to the cloud or not.
  * browser integration: seamless plugin install and integration during setup
  * auto login: no need to click (most websites) on submit button ever again. If you happen to have multiple accounts for a single site you will have an "drop down" list to
    select from.
  * export: csv and its own encrypted .dash format
  * cost: subscription based, annual fee then you can sync your valet to unlimited number of devices including mobile
  * support: good although over email a bit slow
  * security dashboard: gives you warnings about reused passwords, compromised accounts, etc.

#### Dashlane cons:

  * icon view: I hated it, I don't need a thumbnail to identify an account, no detailed "list like" view. To see any details about any accounts, I had to click on them individually
  * no copy item: I had to fill new entry for few similar items even if 2 fields changed only
  * the interface: designed for web accounts only, not so useful for example PIN numbers where there is no username, it simply won't save without username field filled
  * import mismatch: The account name and access URLs are really important for autofill, although I had some websites in the valet after import, on visit I got the popup
    offering account creation which indicated failed match against my valet stored entry
  * idle lock only: no option to lock on screen lock, etc. although it's not so much of a concern for an average user
  * import: KeePass import is Windows only, although works sort of well
  * no virtual keyboard: just some extra protection against key-loggers although not so much of a concern
  * main window: over-engineered, password "categories" does not help to organise accounts. Over 100+ accounts this view becomes overwhelming and confusing
  * apache authentication: incapable of filling / recognising apache like authentication popups
  * security: I didn't like the idea of having the ability to log into their website and browse my password valet (paid only)
  * logo: I just cannot personally associate "antiloop" with password management

#### 1Password pros:

  * offline or online: no built in cloud sync option, Google Drive, iCloud, Dropbox, WiFi or just local valet, it's your choice
  * design: love the application on OSX, similar to KeePass, easy to see details, password strength, etc. in a "detailed like" list view
  * main window: collapsible folder options, excellent organisation and tags, really nice looking interface
  * import: KeePass import allows "field" selection from the CSV export to mach 1Password fields which comes handy for complex entries
  * security: <a href="http://blog.agilebits.com/2013/03/06/you-have-secrets-we-dont-why-our-data-format-is-public/" target="_blank">openly published security design</a>
  * trusted: stable software from vendor with good reputation
  * support: great support and community forums
  * export: csv, text or its own .pif format
  * shared vault: could be very handy between family members
  * security audit: reports weak passwords, old or duplicated entries, etc.

#### 1Password cons:

  * overpriced: no subscription and copies need to be purchased for multiple devices, mobile is also extra
  * Windows version: may be identical functionality wise but looks / feels totally different to OSX. Ugly, clumsy and suggest very early development.
  * browser plugins: install is not part of the setup, needs to be done manually for each browser from various sources and it's convoluted / confusing especially on Windows
  * no auto login: no auto field population either, even a simple browser does that. I don't know why I have to hit "CMD+\" key combination to be able to populate fields and
    log in automatically.
  * no virtual keyboard: just some extra protection against key-loggers although not so much of a concern
  * no 2 factor authentication: and <a href="http://blog.agilebits.com/2011/09/23/two-factor-or-not-two-factor/" target="_blank">it's not even planned</a>
  * browser plugins based on "websockets" and it can break things or requires specific settings on some platforms , Windows especially
  * apache authentication: incapable of filling / recognising apache like authentication popups

#### Verdict

No clear winner for me, decided to keep using KeePass until something better comes along. The big shock for me was the apache server authentication, neither of them could handle
those popups so I was left with "cut n' paste" or "save in the browser", which was a big deal as I have a lot of those.

I liked Dashlane (2.4.1) but it needs refinement, something that 1Password (4.4.1) has on OSX but it is very pricey and I don't think it's justified compared to some of the missing
features/ annoying bits. I also need at least Windows and OSX support, which is done well from Dashlane but 1Password is very much beta on Windows and I am not even sure, on
what grounds they can charge money for it.

I have not evaluated <a href="https://www.lastpass.com" target="_blank">Lastpass</a> yet, at this stage I simply
<a href="http://arstechnica.com/security/2014/07/severe-password-manager-attacks-steal-digital-keys-and-data-en-masse/" target="_blank">do not trust that product</a> and their security practices.

