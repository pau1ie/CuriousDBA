---
title: "Split Tunnelling in Windows 10"
date: 2020-06-12T15:20:49+01:00
tags: ["Network","PowerShell","Windows","VPN"]
---

We use a Virtual Private Network - a VPN to access all our work servers. 
But we use MS teams for meetings,
and our VPN admin asks us not to to push loads of video over the VPN. So what to do?
Disconnecting from the VPN every time a call came in got old pretty fast. And what
do you do if you want to show a colleague something which required VPN access?


## Enabling Split Tunnelling

There was talk of Split tunnelling. This sounded really complicated, and
Windows always seems hard to me, click in the wrong place, and you are doomed
for ever!

Actually it isn't that hard at all!

First of all, make a note of the name of the VPN. This is the name
you gave it when you set it up. Lets say it is "Work VPN". 

Next run PowerShell as administrator. I right click on the Start menu and select
*Windows PowerShell (Admin)* from the list.

To enable split tunnelling, simply run:

```PowerShell
Set-VPNConnection -Name "Work VPN" -SplitTunneling $True
```

To switch it off again, you need to run:

```PowerShell
Set-VPNConnection -Name "Work VPN" -SplitTunneling $False
```

## How it Works

This works really well. To understand how it works, we need to understand what a subnet is.

### What is a Subnet?

IP addresses come in groups. Remember when you used to sit in an office? If you
compared the IP address of your computer to someone sitting near you, the chances
are that the first part would be the same, and only the last number would be
different. Often they would be close together. The computer treats an IP address
as a 32 bit number. They are grouped into subnets by specifying the start of an
IP address, then at the end is a forward slash followed by the number of bits
that are in that subnet. For example: 10.0.0.0/8 means from the 32 bit number:

```
   Bit     : 00000000 01111111 11122222 22222333
   Number  : 12345678 90123456 78901234 56789012
			 
     Binary: 00001010 00000000 00000000 00000000
Hexadecimal:    0   A    0   0    0   0    0   0
    Decimal:       10.       0.       0.       0
```

We only take note of the first 8 bits, i.e. the 10, and any IP address that matches
that number matches the subnet. This way we can tell the computer anything in this subnet
goes this way, and everything else goes that way.

### Showing the routes

To get the computer to show us the routing we can do the following when connected to
the VPN.

```PowerShell
netsh interface ipv4 show route
```

Here is part of mine:
```console
PS C:\WINDOWS\system32> netsh interface ipv4 show route

Publish  Type      Met  Prefix                    Idx  Gateway/Interface Name
-------  --------  ---  ------------------------  ---  ------------------------
No       Manual    0    0.0.0.0/0                   7  192.168.0.1
No       Manual    0    0.0.0.0/0                  18  192.168.0.1
No       Manual    1    10.0.0.0/8                 43  Work VPN
...
No       System    256  127.0.0.0/8                 1  Loopback Pseudo-Interface 1
No       System    256  127.0.0.1/32                1  Loopback Pseudo-Interface 1
No       System    256  127.255.255.255/32          1  Loopback Pseudo-Interface 1
...
```
192.168.0.1 is my home router. I believe what this is saying is, send everything
through my home router apart from the work network which is IP addresses starting 
with 10. This is because the third line says 10.0.0.0/8. Also notice anything starting
127 goes to the Loopback Pseudo-Interface, so 127.0.0.1 would go to my local
computer. I notice there is an additional rule for 127.0.0.1, I am not sure why, this seems
to be the case for a number of routes. There is still a lot I don't understand!

## How To Change the Rules

This works well. Except that we have a test system on a public IP address, but the load
balancer checks the IP address you are coming from before letting you in. So this traffic
also needs to go over  the VPN. 

First I need to  find out the IP address of this
service. I used ping, but there are other ways. Note, this isn't  the real website I
wanted to connect to, it is an example.

```PowerShell
ping www.cam.ac.uk
Reply from 128.232.132.8: Bytes=32 time=24ms TTL=50
...
```

So if I wanted to access this from the VPN I would need to add this IP address to the
route. I need to be connected to the VPN to be able to add the route.

Generally IP addresses come in groups, so it is likely I will want to access
any IP address starting 128.232.132 via the VPN, so I would do the following. This needs
to be done while connected to the VPN:

```PowerShell
netsh interface ipv4 add route 128.232.132.0/24 "Work VPN"
```

And to delete it, one would do:

```PowerShell
netsh interface ipv4 delete route 128.232.132.0/24 "Work VPN"
```

The /24 means match any IP address that matches the first 24 bits of the address.
Looking at the text diagram above, under [What is a Subnet](#what-is-a-subnet) we can see an IP address
is in the subnet if it matches all the numbers (I think proper network geeks call these 
*Octets*) apart from the 0 at the end, which can
be any number.

I can see the IP address has been added in the list below:

```Powershell
PS C:\WINDOWS\system32> netsh interface ipv4 show route

Publish  Type      Met  Prefix                    Idx  Gateway/Interface Name
-------  --------  ---  ------------------------  ---  ------------------------
No       Manual    0    0.0.0.0/0                   7  192.168.0.1
No       Manual    0    0.0.0.0/0                  18  192.168.0.1
No       Manual    1    10.0.0.0/8                 43  Work VPN
...
No       System    256  127.0.0.0/8                 1  Loopback Pseudo-Interface 1
No       System    256  127.0.0.1/32                1  Loopback Pseudo-Interface 1
No       System    256  127.255.255.255/32          1  Loopback Pseudo-Interface 1
No       Manual    256  128.232.132.0/24           43  Work VPN
No       System    256  128.232.132.255/32         43  Work VPN
...
```

Now if I fire up my browser and access the website, it allows me in to the dev
environment, whereas before I would get an error message from the load balancer
telling me I am not allowed access.

## Conclusion

If you aren't scared of PowerShell, there is a lot of flexibility in how you can
route things through the windows VPN. 

## References

I found [this post on Medium](https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy ) 
very helpful along with the 
[Microsoft documentation for netsh](https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy)