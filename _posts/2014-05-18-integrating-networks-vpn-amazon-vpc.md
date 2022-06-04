---
title: Integrating networks over VPN with Amazon VPC
date: 2014-05-18T22:21:58+00:00
author: Iván Ádám Vári
layout: post
permalink: /integrating-networks-vpn-amazon-vpc/
description: Cost effective way of connecting your office networks with pfSense 2.1.3 to Amazon VPC over OpenVPN terminated on an EC2 instance.
keywords: amazon,ec2,vpc,vpn,openvpn,2 way tunnel,routing
categories:
  - AWS
tags:
  - aws
  - cloud
  - vpc
  - ec2
comments: true
sharing: true
footer: true
---
<a href="https://aws.amazon.com/vpc/" target="_blank">Amazon VPC</a> has been out for some time offering full control of isolated local networking in the cloud.
This means that you can have your own private subnet in the cloud, have control over what private IPs your instances are going to use, change the instance type,
should your resource requirements increase and so forth.

This guide is going to be technical, intended for experienced professionals where I will be discussing options and solutions to securely integrate your onsite
(private) LANs with Amazon VPC. It is based on OpenVPN client running on an instance inside VPC, connecting to my remote branch firewall running pfSense 2.1.3
and OpenVPN server. The point-to-point tunnel between the client / server is 2-way, both the client and the server expose their local networks and route traffic
to the other side accordingly. But first, let's take a look at what other option we have.

## Amazon VPC VPN

This is built into VPC and utilizes IPsec which operates on OSI model layer 3 and 4. This technology has been out there for long time, considered fairly secure
and supported on many hardware appliances from most well known vendors. Since it operates on lower layer of the OSI model, it's not intended for end users such as
desktop computers, mobile devices, it's more suitable for connecting networks aka site-to-site VPN.

#### Features and Benefits

  1. Secure. In fact, it was not affected by the recently discovered <a href="http://heartbleed.com" target="_blank">Heartbleed</a> bug of OpenSSL as it's using
     crypto built into the kernel.
  2. Supports dynamic route management with iBGP, suitable for corporate networks with lots of local LANs.
  3. Somewhat easy to implement, widely supported on many hardware platforms, appliances as well as open source software such as
     <a href="https://www.openswan.org" target="_blank">Openswan</a> or <a href="https://www.strongswan.org" target="_blank">strongSwan</a>.
  4. <a href="https://aws.amazon.com/vpc/pricing/" target="_blank">Cheap</a>. Currently it's approximately ~$1 / per day depending on region.
  5. It's really the easiest, simplest, out of box solution at this time of writing.

There is a few good guides out there to get you started, however the easiest and most up-to-date is on
<a href="https://docs.openvpn.net/how-to-tutorialsguides/administration/extending-vpn-connectivity-to-amazon-aws-vpc-using-aws-vpc-vpn-gateway-service" target="_blank">OpenVPN's website</a>.

## VPN terminated on an EC2 Instance inside your VPC

While it does share some features with IPsec, there are a couple of special ones that apply to OpenVPN:

  * Cost savings. Yes, $1 a day is cheap and I have a supported hardware at my branch endpoint, BUT I already have a requirement to run an instance 24/7 inside my VPC
    doing DNS, NTP, git, etc. Adding a VPN and routing functionality on top of those are not hard and just makes sense. I would rather spend that $1 / day on other great
    services such as S3 storage or CloudWatch.
  * Internet availability inside VPC. While the IPsec VPN is superb, it does not provide you internet for your VPC resources, VPC meant to be private, no public traffic
    in / out. So if I have to have an instance doing SNAT for outgoing Internet inside the VPC, then why not terminate the VPN on the same instance?

#### Build your VPC

![Diagram](/assets/img/2014-05/24F4919D-CEFE-48D3-A811-D6BBDC7D2CF9.png)

I will not going into too much detail here, just share the important bits I encountered during my setup. The Amazon guide is well written, please read it for full understanding
of the <a href="http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Scenario2.html" target="_blank">concept</a>.

  * Create a VPC with a preferably /16 mask. eg: `10.0.0.0/16` (Note: CIDR mask must be between /16 and /28)
  * Create at least 2 subnets that overlap with your the VPC CIDR mask, preferably /24. eg: `10.0.0.0/24`, `10.0.1.0/24`. One will be your private LAN with outgoing only Internet,
    the other public with in and outgoing Internet.
  * Create a VPC "Internet Gateway", attach it to your VPC.
  * Ignore "Network ACL" for now, defaults are fine.

Pick a private IP address from your public /24 subnet you created in your VPC. eg: `10.0.0.8` This will be the internal address for your EC2 router. _While you cannot use
static private IP addresses inside VPC at this time of writing, you can associate certain addresses to instances. They still use DHCP but always get the same address, most
likely via MAC address mapping._

  * Associate an "Elastic IP Address" aka public IP for your VPC if you have no free available. Use "network interface" association since we have no instance running yet,
    and use the private IP you picked above. This essentially creates a "network interface" object and adds your selected public / private IPs to it.
  * Create a DHCP option set (optional) if you want to dish out your own DNS servers, domain names, etc. for your VPC instances, but it's not required for the VPN routing.
  * Make sure the security group attached to your VPC has port 22 open for 0.0.0.0/0. This is important initially for being able to reach your newly created "router" instance
    via the public elastic IP until you sort out the internal routing, firewalling, VPN, etc.
  * You will have a default "route" object attached to your VPC, and it will be set to "Main". I recommend to leave that untouched which routes only it's own VPC traffic
    by default. It's important for security reasons because "Main" always applies to a subnets that are not associated with any custom "route" object.

Then, create a custom "route" object, associate that with your "public" VPC subnet where the VPN instance resides. eg: `10.0.0.0/24`. and add your default route to it:

```
0.0.0.0/0          igw-yourid
```

This essentially gives you internet access for outgoing traffic.

>Networks available inside your VPN EC2 instance (aka networks on the far side of your OpenVPN tunnel) will not be available for any other other instance you launch
into this "public" subnet of your VPC. If you need to reach any of those by their internal VPC IP, you will have to add them to the custom "route" object you associated
with the "public" subnet as follows
{: .prompt-info }

``` 
192.168.0.0/24         eni-ID
10.100.100.0/24        eni-ID
10.100.200.0/24        eni-ID
10.100.300.0/24        eni-ID
```
The ID is the "network interface" object you mapped your elastic IP to (attached to your VPN instance). The private class C address is my OpenVPN client network, the other
3 are the LANs behind the pfSense VPN server. You also have to remember, that all instances MUST have elastic IP associated to them inside your "public" VPC subnet, they
will not be able to use the VPN router instance!

Create another custom "route" object, associate it with your "private" VPC subnet eg: `10.0.1.0/24` and add the following route to it:

```
0.0.0.0/0          eni-ID
```

This will add default route to all instances for your "private" VPC subnet via the router EC2 instance. This "route" object can be then associated with any other "private"
subnet you may create later on and required to support outgoing Internet access.

To be able to route any private subnet via your router EC2 instance, you MUST disable source and destination check on the associated network interface object.

```bash
$ aws --region eu-central-1 ec2 modify-network-interface-attribute --network-interface-id "eni-12f235ec" --no-source-dest-check
```

#### Prepare the OpenVPN Server on pfSense

You will need to create a `ccd` entry for your VPC EC2 instance to ensure, that the client always gets the same tunnel IP from the server, as well as for the ability to be
able to publish the remote networks on the client side to the server. It's very simple with the web-ui of pfSense, hence I just show the relevant config entry here, should
you need to set this on console:

```
ifconfig-push 192.168.0.254 192.168.0.253
iroute 10.0.0.0 255.255.255.0
iroute 10.0.1.0 255.255.255.0
```

These routes however required to be added to the relevant server config too (if your run multiple). Again, this is easy on web-ui, relevant config entries are:

```
route 10.0.0.0 255.255.255.0
route 10.0.1.0 255.255.255.0
push "route 10.100.100.0 255.255.255.0"
push "route 10.100.200.0 255.255.255.0"
push "route 10.100.300.0 255.255.255.0"
push "route 10.0.0.0 255.255.255.0"
push "route 10.0.1.0 255.255.255.0"
```

Why the redundant route and iroute statements, you might ask? The reason is that route controls the routing from the kernel to the OpenVPN server (via the TUN interface)
while iroute controls the routing from the OpenVPN server to the remote clients.

And finally, the push-route lines add networks to all VPN clients for all networks available on both side of the tunnel between pfSense and VPC. The reason for this is that
when I connect to the pfSense VPN server with my laptop, I want routes set for both, the office and the VPC networks so I can reach everything while away.

## Launch the Router Instance

The only thing you need to pay attention to:

  * Ensure that you launch the instance to the correct "public" subnet of your VPC
  * Ensure you set the instance to use the prepared "network interface" object


Once running, you should be able to reach it via SSH protocol over its public address. Depending on your distro of choice, you may need to turn on routing if not enabled by
default in the kernel:

```bash
$ sysctl -w net.ipv4.ip_forward=1
```

OR

```bash
$ echo 1 > /proc/sys/net/ipv4/ip_forward
```

Install then configure your OpenVPN client, set it to run at boot, start, etc. Note: there is nothing specific you need to configure as long as you run client here.
Add a couple of iptables rules to NAT traffic coming from your VPC destined to the Internet:

```bash
$ iptables -A POSTROUTING -s 10.0.0.0/16 -d 10.0.0.0/16 -m comment --comment "VPC->VPC:SKIP" -j ACCEPT
$ iptables -A POSTROUTING -s 10.0.0.0/16 -d 10.100.0.0/16 -m comment --comment "AWS->REMOTE SUBNETS:SKIP" -j ACCEPT
$ iptables -A POSTROUTING -o eth0 -s 10.0.1.0/24 -m comment --comment "SUBNET-PRIVATE->INTERNET:MASQ" -j SNAT --to-source 10.0.0.8
```

We need the ACCEPT rules to avoid rewriting traffic between VPC subnets as well as between VPC and remote subnets, otherwise they have no affect. Make sure you save the
rules and enable firewall service to run during boot. Note: this config assumes you have NO other firewall rules configured, those should be added when you have working
routing.

You should be able to reach your instance after all this via it's private address. When you have a working setup, remove the SSH entry from the "security group" of your
VPC for additional safety.
