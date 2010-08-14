---
layout: article
title: IP Address Basics
categories: network
updated_at: 2010-07-30
---

Goal
====

Explain the basics of networking to a person familiar with a Western-US-style grid system such that he can properly set the ip address of a gumstix overo manually when dhcp is not available.

It's important to understand not just how to set up one gumstix, but other network devices that are laying around might need to be set up too, hence the explanation.

WARNING: This needs major revision before it meets that goal.

An IP Address is like a Street Address
==========

If you've lived in the west you may be familiar with the grid system.
There's *Center St*, *Main St*, and perhaps **State St** or *University Ave* in every city.
From *Center St* and north all streets are called Nth North. Likewise south of center are all Nth South.
East and West of main all streets are called Nth East and Nth West.

Likewise, ip addresses tell one computer where it can find another.

Common IP addresses

  * 10.x.x.yyy - large private network
  * 192.168.x.yyy - small private network
  * 169.254.yyy.yyy - automatic network (no internet access)
  * 127.0.0.1 - "localhost" - the non-functional loopback network address for testing and compatibility
  * yyy.yyy.yyy.yyy - internet address


A Gateway is like State St (in the west anyway)
=======

In the west you can get (almost) anywhere in the city without needing a map.
You may need directions to get from one city to the next (or to the interstate), but often you can count on *State St* to take you there.

The gateway is somewhat like *State St*, it helps you get to the next network over - such as the large **internet**.

Common Gateway address

    * 10.x.x.1 - large private network
    * 192.168.x.1 - small private network
    * yyy.yyy.yyy.yyy - public network
    * automatic networks have no gateway

**A computer which does not connect to the internet does not have a gateway**


Netmask is like city limits
=======

Let's say you're 2 miles into your 30 mile trip to get to a certain 2900 N (along State St rather than the interstate - for example's sake).
You just passed a 2500 N and all of the sudden you're at 1700 S. What!? Weren't you trying to get to 2900 N?

You've crossed the city limit! But that's okay, right? The 2900 N you're looking for is in a city another 28 miles away.


A netmask is like a city limit. It tells the computer how far it can expect to be able to search before going to the gateway.

Literally, the netmask is the value that gets bitwise `AND`ed with the ip of the computer to determine the network size.

Common Netmasks

    * 255.255.255.0 - small private network
    * 255.255.0.0 - large private network, automatic network
    * 255.255.255.240 - 16 leased public (internet) addresses in a multi-office complex

255.255.0.0 is almost always a safe value if you're not sure what to put. There are plenty of [Online Network Calculators](http://jodies.de/ipcalc?host=192.168.0.50&mask1=28&mask2=) to help with advanced settings.


DNS is like a GPS
=================

For those of you from the east, you might find it difficult to tell Tom that Dick lives on 200 E, 300 N... or was it 300 E, 200 N...
(people from the west are all too familiar with that problem)
anyway, you'd remember Jefferson Ave and 15 Sidney Lane a lot easier if Dick lived at that address.
Harry's GPS would allow you to put in easy-to-remember coordinates and give back exact grid-system coordinates.

www.google.com is like Jefferson Ave. 74.125.19.100 is like 200 E, 300N.

DNS is like GPS that translates web names to addresses.

Lucky for us, the only two DNS we need are both gri-system and easy-to-remember (too bad they aren't the pre-loaded defaults)

  * 8.8.8.8
  * 8.8.4.4


Example 1
========

Let's say that we have a small public network in a multi-office complex. For this senario we'll pretend that we're the cool kids at the googleplex.

  * We know that we have 16 addresses on our network
  * The whiteboard shows that 74.125.19.75 is not in use so we want to use that to put a test system public and live.
  * The netmask would be something like 255.255.0.0 by default, but we know that really it's 255.255.255.240 (because we only have 16 addresses)
  * The gateway would probably be 74.125.19.65 - the first available address on the network (just like 192.168.1.1, but in a higher subnet)
  * We want our DNS to be 8.8.8.8 (always)

In an ad-hoc (transient) configuration that it lost on reboot or when the network cable is unplugged:

    # We have a working network card
    ping -c 1 127.0.0.1 >/dev/null
    
    # We're on a network with this gateway
    ifconfig eth0 74.125.19.75 netmask 255.255.255.240 up
    ping -c 1 74.125.19.65 >/dev/null
    
    # Our gateway takes us to the internet
    route add default gw 74.125.19.65
    ping -c 1 8.8.8.8 >/dev/null
    
    # We can resolve human-readable addresses
    sh -c 'echo "nameserver 8.8.8.8" >> /etc/resolv.conf'
    sh -c 'echo "nameserver 8.8.4.4" >> /etc/resolv.conf'
    ping -c 1 www.google.com >/dev/null

In a more permanent configuration:

`/etc/network/interfaces`

    auto eth0 eth0:0 eth0:3
    # no DHCP, no static IP. Great OOBE
    iface eth0 inet static
      address 169.254.0.10
      netmask 255.255.0.0

    # what customers are already familiar with
    iface eth0:0 inet static
      address 192.168.1.10
      netmask 255.255.255.0

    # an address to use for testing on the internet
    iface eth0:3 inet static
      address 74.125.19.77
      netmask 255.255.255.240
      network 74.125.19.64
      broadcast 74.125.19.79
      gateway 74.125.19.65
    

Example 2
=========

A computer with the address of 192.168.254.53 and a netmask of 255.255.0.0 knows that it can only find computers within the 192.168.yyy.yyy network.
If the netmask were 255.255.255.0 it would only look within 192.168.254.yyy. If the netmask were 255.255.255.240 it would assume that it needed to go
to the gateway to reach any computer not between 192.168.254.49 and 192.168.254.62. The network would be literally 192.168.254.64 and the broadcast
would be 192.168.254.80. In each subnet you lose 2 addresses.

Your gateway would most likely be 192.168.254.65 - the first available address on the subnet.

