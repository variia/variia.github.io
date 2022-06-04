---
title: WD MyCloud 2T NAS Review
date: 2014-08-03T22:38:52+00:00
author: Iván Ádám Vári
layout: post
permalink: /wd-mycloud-2t-nas-review/
description: After my Seagate Central died, I got given an alternative from WD. This is my quick and dirty WD MyCloud 2T NAS review and personal experience.
keywords: wd,mycloud,review
categories:
  - Hardware
tags:
  - hardware
  - nas
  - wd
comments: true
sharing: true
footer: true
---
I purchased a <a href="http://www.seagate.com/external-hard-drives/home-entertainment/media-sharing-devices/seagate-central" target="_blank">Seagate Central</a> 2T NAS 5 months
ago, for a low cost home media center solution. It worked reasonably well considering the low ~US130 cost although, I had ongoing issues with firmware updates, occasional drive
performance, etc. Unfortunately, it failed last week and while I was looking for alternatives, I learnt that I was not the only one having problems with that device so I simply
lost trust in Seagate forever.

I returned the drive, and the store offered me the <a href="http://www.wdc.com/mycloud/" target="_blank">WD MyCloud</a> 2T as a replacement alternative without extra cost what
I happily accepted.

## WD MyCloud 2T NAS Review

This is my personal opinion and experience compared to Seagate Central, not necessarily an official "review" of the hardware and its provided features.

#### Design

The enclosure is very solid, fairly compact and feels good. I find the upright design a bit impractical as well as unstable compared to the Seagate Central and I have to
admit, I am not a big fan of the white housing and the blue LED in front but it's just personal preference. Comparing to Seagate Central, the device feels actually a lot
cooler, it's more like hand warm rather than hot.

#### Features:

  * 1 x Gbit interface
  * 1 x USB 3.0 port
  * Dual Core processor
  * DLNA 1.5 and UPnP Certified
  * iTunes support
  * TimeMachine support
  * Mobile apps
  * Remote access

## The GOOD:

#### Software:

The WebUI is very nice, clean and both easy to read and drive. The front page is well organised, dashboard alike but not "too busy" and easy to understand even for an average
user but still giving you the option for detailed information, should you need it.

![Home](/assets/img/2014-08/5347607A-0F91-4454-84CC-85F0DDD1F34C.jpg)

#### Shares:

![Shares](/assets/img/2014-08/1781162B-5656-424A-A26C-3A9B6CCE5285.jpg)

Shares can have individual access settings or set public (accessible without password) as well as set to be excluded from media serving. (does NOT include exclusion from media
scan unfortunately)

#### Notifications:

![Notifiations](/assets/img/2014-08/C6ABB8A2-A36D-4CE0-A4ED-93A93A7549F3.jpg)

You can set up email notifications and their importance. It's not so much of a monitoring but certainly helps to keep you up to date about the events occurring on the drive
including predictive failure, reboots, etc.

#### Settings:

![Settings](/assets/img/2014-08/23FBE529-56A2-45DB-ACF7-58AEF7DB125F.jpg)

Not so much of a feature, but it was great to see (after Seagate Central) that I have the ability to backup my configuration. Unfortunately, it only backs up the basics like
network settings but it does not save some of the personal preferences such as "Cloud Access" settings, etc.

#### Network:

![Network](/assets/img/2014-08/0DEE40B5-9495-4540-B77E-AAA6D3B21887.jpg)

One of the biggest surprise (positive) for me was the fact that this device allows SSH access as well as FTP access. Furthermore, the access is root which may not be good for
some but advanced users can really leverage this functionality especially when disaster strikes.

#### Safepoints:

![Safepoints](/assets/img/2014-08/9BA06E47-3305-42D5-9581-434A0FCD6B6F.jpg)

Basically, you can set up another device and create a snapshot or mirror of your data at any given point of time. Considering the sizes available, I am not sure how feasible
this is or how long it takes to copy 2T-4T over the cable even on Gbit network.

Having the ability to clone one drive to another is not a bad idea and if you are looking for safety and value, it is cheaper to get 2 of these than one
[MyCloud EX](http://www.wdc.com/mycloudex2/) with 2 disks. Although that would give you encryption support as well as continuos operation even when one of the drive fails
(assuming RAID1 setup).

#### DLNA 1.5 and Twonky:

Both my Samsung BluRay player and TV can stream seamlessly from the device. Just finished updating the firmware and was delighted to see a fairly recent Twonky release 7.2.8
(this time of writing) included in the image.

#### Under the hood:

It's running Debian Linux 7, data volumes feature EXT4 filesystem on dual core ARM platform.

```bash
Linux WDMyCloud 3.2.26 #1 SMP Tue Jun 17 15:53:22 PDT 2014 wd-2.2-rel armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.

WDMyCloud:~# cat /proc/cpuinfo
Processor       : ARMv7 Processor rev 1 (v7l)
processor       : 0
BogoMIPS        : 1292.69

processor       : 1
BogoMIPS        : 1292.69

Features        : swp half thumb fastmult vfp edsp neon vfpv3 tls
CPU implementer : 0x41
CPU architecture: 7
CPU variant     : 0x2
CPU part        : 0xc09
CPU revision    : 1

Hardware        : Comcerto 2000 EVM
Revision        : 0001
Serial          : 0000000000000000

WDMyCloud:~# free
             total       used       free     shared    buffers     cached
Mem:        232448     170816      61632          0       7680      38784
-/+ buffers/cache:     124352     108096
Swap:       500672      94208     406464

WDMyCloud:~# mount
/dev/root on / type ext3 (rw,relatime,errors=continue,barrier=1,data=ordered)
tmpfs on /run type tmpfs (rw,nosuid,noexec,relatime,size=23296k,mode=755)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=40960k)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,relatime,size=10240k,mode=755)
tmpfs on /run/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620)
fusectl on /sys/fs/fuse/connections type fusectl (rw,relatime)
tmpfs on /tmp type tmpfs (rw,relatime,size=102400k)
/dev/root on /var/log.hdd type ext3 (rw,relatime,errors=continue,barrier=1,data=ordered)
ramlog-tmpfs on /var/log type tmpfs (rw,relatime,size=20480k)
/dev/sda4 on /DataVolume type ext4 (rw,noatime,nodiratime,user_xattr,barrier=0,data=writeback)
/dev/sda4 on /CacheVolume type ext4 (rw,noatime,nodiratime,user_xattr,barrier=0,data=writeback)
/dev/sda4 on /nfs/TimeMachineBackup type ext4 (rw,noatime,nodiratime,user_xattr,barrier=0,data=writeback)
/dev/sda4 on /nfs/Public type ext4 (rw,noatime,nodiratime,user_xattr,barrier=0,data=writeback)
/dev/sda4 on /nfs/SmartWare type ext4 (rw,noatime,nodiratime,user_xattr,barrier=0,data=writeback)
/dev/sda4 on /nfs/vari type ext4 (rw,noatime,nodiratime,user_xattr,barrier=0,data=writeback)
nfsd on /proc/fs/nfsd type nfsd (rw,relatime)
```

## The BAD:

#### Remote Access:

I never liked this idea, never meant to use it but you can access your content over the internet after registering your user on <a href="https://www.wd2go.com/" target="_blank">wdmycloud.com</a>.
I was wondering about why it is and how my content is made available, whether it's encrypted, etc. After logging in via SSH, I discovered a process running on the device:

`root 1585 2.0 1.5 6464 3712 ? SN 07:48 0:00 openvpn /usr/local/orion/openvpnclient/client.ovpn`

When you disable "Cloud Access" under "Settings -> General", this process disappears so generally, I was happy to see that personal content is served encrypted over the Internet.
However, I see some serious privacy issues around using OpenVPN for this feature.

How it is configured:

```bash
WDMyCloud:~# grep . /usr/local/orion/openvpnclient/client.ovpn | grep -vE ';|#'
client
dev tun
proto udp
remote orionrelaya37.wd2go.com 14187
remote orionrelaya42.wd2go.com 14418
remote orionrelaya13.wd2go.com 14508
remote orionrelaya52.wd2go.com 14603
remote orionrelaya1.wd2go.com 14299
remote orionrelaya67.wd2go.com 14789
remote orionrelaya16.wd2go.com 14849
remote orionrelaya15.wd2go.com 14264
remote orionrelaya56.wd2go.com 14104
remote orionrelaya24.wd2go.com 14441
remote orionrelaya38.wd2go.com 14711
remote orionrelaya4.wd2go.com 14740
remote orionrelaya63.wd2go.com 14775
remote orionrelaya74.wd2go.com 14935
remote orionrelaya96.wd2go.com 14097
remote orionrelaya97.wd2go.com 14807
remote orionrelaya70.wd2go.com 14592
remote orionrelaya65.wd2go.com 14777
remote orionrelaya26.wd2go.com 14076
remote orionrelaya79.wd2go.com 14258
remote orionrelaya25.wd2go.com 14080
remote orionrelaya62.wd2go.com 14285
remote orionrelaya6.wd2go.com 14198
remote orionrelaya17.wd2go.com 14450
remote orionrelaya18.wd2go.com 14703
remote orionrelaya75.wd2go.com 14405
remote orionrelaya51.wd2go.com 14442
remote orionrelaya28.wd2go.com 14474
remote orionrelaya88.wd2go.com 14041
remote orionrelaya21.wd2go.com 14080
remote orionrelaya35.wd2go.com 14174
remote orionrelaya66.wd2go.com 14824
remote orionrelaya49.wd2go.com 14745
remote orionrelaya73.wd2go.com 14091
remote orionrelaya53.wd2go.com 14688
remote orionrelaya61.wd2go.com 14884
remote orionrelaya47.wd2go.com 14858
remote orionrelaya32.wd2go.com 14785
remote orionrelaya76.wd2go.com 14217
remote orionrelaya41.wd2go.com 14183
remote orionrelaya72.wd2go.com 14173
remote orionrelaya78.wd2go.com 14709
remote orionrelaya7.wd2go.com 14140
remote orionrelaya98.wd2go.com 14744
remote orionrelaya22.wd2go.com 14257
remote orionrelaya64.wd2go.com 14021
remote orionrelaya69.wd2go.com 14003
remote orionrelaya48.wd2go.com 14064
remote orionrelaya71.wd2go.com 14951
remote orionrelaya94.wd2go.com 14567
tls-exit
explicit-exit-notify 3
script-security 2
up "/bin/rm -f /var/log/messages &&"
echo "Initialization Sequence Completed"
remote-random
resolv-retry infinite
nobind
persist-key
persist-tun
ca /usr/local/orion/openvpnclient/ca.crt
ns-cert-type server
verb 3
sndbuf 262144
--auth-user-pass /usr/local/orion/openvpnclient/auth.txt
reneg-sec 0
```

In nutshell: when you enable "Cloud Access", you start a VPN client which "connects" to one of those "orion" WD endpoints you see configured as "remote" and build a point-to-point
encrypted tunnel with it. This brings up a new virtual "tun" interface on your device with a specific class A private IP address, in my case:

```
 tun0 Link encap:UNSPEC HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
 inet addr:10.8.65.85 P-t-P:10.8.0.1 Mask:255.255.255.255
 UP POINTOPOINT RUNNING NOARP MULTICAST MTU:1500 Metric:1
 RX packets:0 errors:0 dropped:0 overruns:0 frame:0
 TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
 collisions:0 txqueuelen:100
 RX bytes:0 (0.0 B) TX bytes:0 (0.0 B)
```

This basically exposes your ENTIRE device to the "other" end you established the VPN tunnel with. The system is a cut-down version of Linux, it does not have firewall filtering
or other security software built into it. Considering that FTP and SSH running on your device, anybody from the other end of the tunnel has the ability to log into your device
and do whatever they want, including browsing your content.

While I understand that it is a cost effective way of protecting your own content from 3rd parties, it does not protect you from the vendor itself so if you want to get access
to your content while away, you will have to trust WD some ways.

#### AFP performance:

Bad news for OSX users, even listing a decent directory with few hundred files takes a long time. I could not be bothered too much about it,
<a href="http://support.apple.com/kb/HT5884" target="_blank">OSX Mavericks now defaults to SMB</a> so Apple clearly signaled that it's moving away from its in-house developed protocol.

Using the device over SMB (cifs) is fine, although I have not upgraded to OSX Mavericks yet so my experience is based on OSX Mountain Lion 10.8.5.

#### Thumbnails and the mysterious .wdmc directories:

If you log into the device via SSH or browse it with AFP, you will see mysterious `.wdmc` folders all over the place with lots of tiny image files in them. They are thumbnails
of your media files so every folder that has media (image, video) in it will have one of these.

SMB (samba) hides these so you won't see these normally but the issue with this service is that they are created regardless you want them or not. I am an amateur photographer,
have thousands of images and these are not only take space from the device, but also expensive to create and manage. When you upload new content such as image, the device will
display "Content Scan" "Building", that means it's scanning and polluting your drive with these and depending on the volume, it can take a long long time and your drive performance
will be degraded.

I was under the impression, that if I turn media serving off for my "photo share" it would be ignored. I actually like these thumbnails for my video share" what I browse over
DLNA but I don't want it for everything.

So until WD kindly gives us the WebUI option to turn this off, you can do this (advanced users only):

```bash
$ /etc/init.d/wdmcserverd stop
$ /etc/init.d/wdphotodbmergerd stop
$ update-rc.d wdphotodbmergerd disable
$ update-rc.d wdmcserverd disable
```

That will just disable the service, it will not remove the directories and this most likely to be required again after firmware updates.

