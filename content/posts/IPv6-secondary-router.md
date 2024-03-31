---
title: "IPv6 Secondary Router"
date: 2024-03-31T19:49:00+08:00
tags: [Linux, Networking, Clash]
---

## Why secondary router

A secondary router may help setting up a transparant proxy for a subset of the devices in a LAN.

## IPv4

In the world of IPv4, almost all LANs have a router running a DHCP server to assign LAN IPv4 addresses. With DHCPv4, the router can specify a default gateway for a given subnet. This effectively gives us the capability to route traffic from specific devices to any device on the LAN.

## IPv6

In the world of IPv6, there is no need to keep a stateful private LAN address pool thanks to the huge address space of IPv6. That is, stateless address autoconfiguration (SLACC) is preferred. Despite there is DHCPv6, the fundamental mechanism Router Advertisement, RA (or Neighbor Discovery Protocol, NDP) doesn't support default router (gateway) like DHCPv4. The sender of the RA messages naturally becomes the router for all the clients that accepts the RA.

So, the secondary router is still up and running. How can we route traffic from a specific set of devices to the transparant proxy on the secondary router in the world of IPv6?

## A solution

Fortunately, the well-known RA implementation on Linux, [RADVD](https://linux.die.net/man/5/radvd.conf), supports `AdvDefaultPreference` which can set the RA message preference level. We can simply keep the primary router as is. Then, config a RADVD on the secondary router to send RA message with a higher preference.

There are some caveats:
1. How to keep the prefix from the primary router?
    a. In the `prefix` section of an interface, set an all-zero (e.g. `::/64`) prefix configures RADVD to advertise what ever that interface gets from the primary router.
2. How to hijack just a given set of devices in the LAN?
    a. In the `clients` section of an interface, name the clients you want to route traffic from by their [link-local address](https://en.wikipedia.org/wiki/Link-local_address?useskin=vector). This configures RADVD to send RA to the list of unicase addresses.
